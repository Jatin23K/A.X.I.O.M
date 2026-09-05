# A.X.I.O.M. — Module 06: Observability & RAG Triad Telemetry
## Groundedness SLA (1.00) · OpenTelemetry Distributed Traces · Safe Failure Contracts · Verbatim Citations

> **Elevator Summary:** In production enterprise RAG systems, a confident hallucination is far more damaging than a declared knowledge gap. A.X.I.O.M. guarantees zero hallucinations through three interlocking mechanisms: the **RAG Triad** (continuous mathematical scoring of Context Relevance, Groundedness, and Answer Relevance), the **Safe Failure Contract** (proactively blocking LLM generation when retrieved context is insufficient), and the **Verbatim Citation Engine** (tagging every factual claim with `[Ref X]` pointing directly to exact PDF pages, audio timestamps, or video frames). Telemetry is captured via OpenTelemetry distributed tracing without adding synchronous latency to client responses.

---

## 1. The RAG Triad Quality Framework

A.X.I.O.M. continuously computes three foundational quality metrics on every generation cycle:

### Metric 1: Context Relevance (Retrieval Precision)
* **Definition:** Evaluates the proportion of retrieved chunks that contain factual information relevant to answering the prompt.
$$\text{Context Relevance} = \frac{|\text{Relevant Chunks Retrieved}|}{|\text{Total Chunks Retrieved}|}$$
* **Production SLA:** Target $> 0.70$ | **Safe Failure Floor:** $< 0.40$
* **Failure Remediation:** If Context Relevance falls below $0.40$, vector search failed to retrieve authoritative context. The pipeline triggers Safe Failure Gate 2, bypassing the LLM to return a Knowledge Gap disclosure.

### Metric 2: Groundedness / Faithfulness (Hallucination Suppression)
* **Definition:** Verifies that every assertion, numerical quantity, date, or claim made in the generated answer is directly entailed by the retrieved context.
$$\text{Groundedness} = \frac{|\text{LLM Assertions Supported by Context}|}{|\text{Total LLM Assertions}|}$$
* **Production SLA:** **1.00 Target** (Zero-tolerance policy for unbacked claims).
* **Failure Remediation:** If Groundedness $< 1.00$, Output Guardrails strip the unverified sentences and append a citation gap alert.

### Metric 3: Answer Relevance (Semantic Intent Alignment)
* **Definition:** Verifies that the generated response directly answers the user's inquiry rather than digressing into tangential topics.
$$\text{Answer Relevance} = \text{CosineSimilarity}(\mathbf{v}_{\text{query}}, \mathbf{v}_{\text{answer}}) \ge 0.85$$
* **Production SLA:** Target $\ge 0.85$.

---

## 2. OpenTelemetry Distributed Trace Topology

Every request is assigned a globally unique `Trace_ID` (e.g. `TRC_4f6eea4f`) propagated across the entire pipeline. The execution spans provide end-to-end visibility into latency bottlenecks, token consumption, and cost:

```text
Trace_ID: 'TRC_4f6eea4f'
  │
  ├── Span 1: gateway_jwt_auth       duration: 0.3ms
  ├── Span 2: input_guardrails        duration: 3.1ms
  ├── Span 3: redis_cache_lookup      duration: 12.4ms  → CACHE_MISS
  ├── Span 4: qdrant_prefilter        duration: 18.7ms
  ├── Span 5: modality_router         duration: 0.5ms   → TEXT
  ├── Span 6: matryoshka_embed        duration: 45.2ms
  ├── Span 7: rrf_hybrid_search       duration: 38.1ms
  ├── Span 8: cross_encoder_rerank    duration: 82.3ms
  ├── Span 9: gemini_flash_inference  duration: 612.0ms (Primary Compute)
  ├── Span 10: output_guardrails      duration: 22.4ms
  ├── Span 11: citation_mapping       duration: 8.2ms
  └── Span 12: redis_cache_write      duration: 4.1ms (Async Background)

Total Execution: 847.3ms | RAG Triad: CR=0.92, G=1.00, AR=0.91 | Total Cost: $0.0014
```

---

## 3. Safe Failure Decision Boundaries

```text
[ Trigger 1: Context Starvation ]
  Context Relevance < 0.40 ──► Block LLM Call ──► Return: "A.X.I.O.M. has no verified documentation on this topic."

[ Trigger 2: Groundedness Violation ]
  Groundedness < 1.00 ──► Output Guardrail ──► Strip unbacked sentences; present only verified factual claims.

[ Trigger 3: Upstream Provider Fault ]
  HTTP 429 / 503 / Timeout ──► Circuit Breaker ──► Instant fallback to secondary tier (Never serve 500 error).
```

---

## 4. Verbatim Citation Mapping Engine

Enterprise users cannot rely on unreferenced summaries. A.X.I.O.M. tags every factual sentence with an immutable citation anchor `[Ref X]` mapping to exact physical artifacts:

```text
[Ref 1] ──► PDF Document: "Q2_Financial_Audit.pdf", Page 42, Paragraph 3
[Ref 2] ──► Audio Recording: "Executive_AllHands.wav", Timestamp: 14:32 - 14:55
[Ref 3] ──► Video Keyframe: "Turbine_Inspection.mp4", Frame #4520 (Timestamp: 02:30.6)
```
