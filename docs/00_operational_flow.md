# A.X.I.O.M. — Module 00: Complete Operational Flow
## End-to-End Request Pipeline: HTTP Request to Grounded Response

> **Elevator Summary:** Every incoming user query traverses an 11-stage sequential pipeline with hard latency budgets, cryptographic security verification, and quality gates. The architecture guarantees Cache Hits in $<15	ext{ms}$, Text RAG in $<600	ext{ms}$, and Omnimodal Video RAG in $<2.5	ext{s}$ — with zero hallucinations enforced via the RAG Triad Observability layer.

---

## 1. Full Pipeline Architecture Diagram

`mermaid
sequenceDiagram
    autonumber
    actor User as Client / User
    participant Gateway as 1. API Gateway (JWT Auth)
    participant GuardIn as 2. Input Guardrails
    participant Cache as 3. Role-Scoped Redis Cache
    participant Qdrant as 4. Qdrant HNSW Pre-Filter
    participant Router as 5. Modality Router
    participant ModelRouter as 6. Department Model Router
    participant LLM as 7. LLM / VLM Generation
    participant GuardOut as 8. Output Guardrails
    participant Telemetry as 9. RAG Triad Telemetry (Async)
    participant CacheWrite as 10. Cache Write (Async)

    User->>Gateway: HTTP Request (Bearer JWT + Query + Modality)
    alt Invalid Token / Expired
        Gateway-->>User: 401 Unauthorized (<0.5ms)
    end
    Gateway->>GuardIn: Validate Tenant, Dept, Clearance (<0.5ms)
    alt Prompt Injection / PII Detected
        GuardIn-->>User: 400 Bad Request (<5ms)
    end
    GuardIn->>Cache: Role-Scoped Key Lookup (<15ms)
    alt Cache HIT (L1 Exact SHA-256 or L2 Vector >= 0.92)
        Cache-->>User: Return Instant Cached Response (<15ms)
    else Cache MISS
        Cache->>Qdrant: Execute HNSW Pre-Filter (RBAC Checkpoint 2) (<20ms)
        Qdrant->>Router: Retrieve Filtered Candidate Chunks
        Router->>ModelRouter: Multi-Vector / Hybrid RRF Context Pack
        ModelRouter->>LLM: Cost-Optimized Route (Flash-Lite / Flash / Pro)
        LLM->>GuardOut: Raw Synthesized Response + Citations
        alt Groundedness Score < 1.00 (Unbacked Claims)
            GuardOut-->>User: Safe Failure Warning (Citation Gap)
        else Grounded Response Passed
            GuardOut-->>User: Grounded Answer + [Ref X] Citations
            par Asynchronous Telemetry & Cache Write
                GuardOut-)Telemetry: Async OpenTelemetry Trace & RAG Triad Logging
                GuardOut-)CacheWrite: Store Verified Payload with 24h TTL
            end
        end
    end
`

---

## 2. The 11 Sequential Execution Stages

| Stage | Subsystem | Target Latency | SLA Max | Primary Architectural Function |
| :--- | :--- | :---: | :---: | :--- |
| **01** | **API Gateway** | < 0.5ms | 1.0ms | Decodes JWT Bearer token; extracts Tenant_ID, Department, Clearance_Level; computes Permission_Hash = SHA256(Tenant + Dept + Clearance). Rejects invalid tokens with HTTP 401. |
| **02** | **Input Guardrails** | < 5.0ms | 10.0ms | Regex & classifier sanitization: strips PII, blocks adversarial jailbreaks/prompt injection, and enforces character bounds. |
| **03** | **Role-Scoped Cache** | < 15.0ms | 30.0ms | Checks L1 Exact Hash (SHA-256) and L2 Semantic Vector Similarity ($\ge 0.92$) in Redis RAM. On hit, bypasses vector search and LLM generation entirely. |
| **04** | **Qdrant Pre-Filter** | < 20.0ms | 50.0ms | Enforces Checkpoint 2 RBAC directly inside Qdrant HNSW graph before similarity calculations: 
bac_roles CONTAINS user_role AND 
equired_clearance <= user_clearance. |
| **05** | **Modality Router** | < 1.0ms | 5.0ms | Directs query to specialized retrieval engines: Text (Matryoshka + BM25 RRF), Image (ColPali ViT MaxSim), Audio (WhisperX + CLAP), or Video (3-Stream Temporal). |
| **06** | **Department Router** | < 0.5ms | 2.0ms | Directs context to optimal intelligence tier: HR/Support $	o$ Gemini Flash-Lite; Engineering $	o$ Flash; Legal/Finance $	o$ Pro (88.6% cost reduction). |
| **07** | **LLM/VLM Generation** | < 600ms | 2500ms | Synthesizes answers under strict 8,000-token context budget. Governed by a 6-key round-robin pool with stateful circuit breakers. |
| **08** | **Output Guardrails** | < 50.0ms | 100.0ms | Verifies hallucination-free output. Computes Faithfulness. If Groundedness $< 1.00$, triggers Safe Failure contract to suppress unverified claims. |
| **09** | **RAG Triad Scoring** | *Async* | *Async* | Non-blocking telemetry: evaluates Context Relevance, Groundedness, and Answer Relevance. Emits OpenTelemetry traces without blocking response delivery. |
| **10** | **Cache Write** | *Async* | *Async* | Stores verified, pre-sanitized answer payload in Redis RAM under the caller's role-scoped key with a 24-hour TTL. |
| **11** | **Client Delivery** | < 1.0ms | 5.0ms | Returns structured response with verified inline [Ref X] citations, document links, and precise audio/video playback timestamps. |

---

## 3. Safe Failure Decision Gates

In enterprise mission-critical deployments, **a confident wrong answer is catastrophic**. A.X.I.O.M. introduces deterministic circuit breakers at four failure boundaries:

`
[ Gate 1: Authentication ] ──► Invalid / Expired JWT ──────────────► HTTP 401 Unauthorized (Zero data exposed)
[ Gate 2: Retrieval ]      ──► Context Relevance < 0.40 ──────────► Block LLM generation; return Knowledge Gap notice
[ Gate 3: Generation ]     ──► Groundedness Score < 1.00 ──────────► Suppress hallucinated claims; return partial cited facts
[ Gate 4: Infrastructure ] ──► Upstream API HTTP 429 / 503 ────────► Instant fallback to Flash-Lite or local Ollama
`

---

## 4. End-to-End Latency Summary

`
Cache Hit (L1/L2):      [========] ~15ms (99% latency reduction)
Text RAG (Full):        [================================] ~450ms - 600ms
Visual RAG (ColPali):   [============================================] ~750ms - 1100ms
Video 3-Stream RAG:     [================================================================] ~1800ms - 2400ms
`
