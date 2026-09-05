# A.X.I.O.M. — Module 05: Ingestion ETL & Stale Index Purge
## Asynchronous CQRS Write Pipeline · Soft-Delete Versioning · Department Retention Policies

> **Elevator Summary:** A.X.I.O.M. completely decouples its data lifecycle into two independent architectural paths: the **Read Side** (live user query serving with $<15\text{ms}$ cache and $<600\text{ms}$ RAG SLAs) and the **Write Side** (background document ingestion, embedding generation, and index management). This CQRS (Command Query Responsibility Segregation) pattern guarantees that ingesting 500-page enterprise PDFs or 4-hour video tutorials never causes latency spikes or blocks live user queries. Document lifecycles are governed by a 3-state Soft-Delete protocol and decentralized Department Retention Policies.

---

## 1. CQRS Architecture: Decoupled Read & Write Paths

```text
┌─────────────────────────────────────────────────────────┐
│              WRITE SIDE (Async Background)              │
│  Admin Upload → Celery Task Queue → Worker Pipeline     │
│  Extract → Transform → Vectorize → Load → Cache Purge   │
│  Primary Goal: Zero Index Fragmentation, Data Freshness │
└────────────────────────────┬────────────────────────────┘
                             │ Writes to
                    ┌────────▼────────┐
                    │  Qdrant + Redis │
                    └────────┬────────┘
                             │ Reads from
┌────────────────────────────▼────────────────────────────┐
│               READ SIDE (Synchronous Live)              │
│  Client Query → API Gateway → Cache → RAG → LLM Output  │
│  Primary Goal: Sub-15ms Cache Hits, Hard SLAs (<600ms)  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. The 8-Stage Asynchronous ETL Pipeline

When a new document or media file is submitted, the ingestion engine executes asynchronously:

1. **HTTP 202 Accepted:** Admin uploads file via REST API. The gateway immediately returns `HTTP 202 Accepted` with a tracking `task_id`. The client is never blocked.
2. **Task Queue Ingestion:** Celery worker picks up the job from an in-memory Redis message queue.
3. **EXTRACT (Modality-Specific Parsing):**
   * Video: PyAV / FFmpeg demuxing into visual scene frames and audio streams.
   * Audio: WhisperX forced alignment speech extraction + CLAP acoustic feature extraction.
   * Image / PDF: High-resolution rasterization to $1024 \times 1024$ RGB tensors for ColPali ViT patches.
   * Text: Markdown, HTML, and structured document parsing.
4. **TRANSFORM (Chunking & Embedding):**
   * Computes atomic chunks (Parent-Child text, VAD speech windows, scene keyframes).
   * Generates dense vectors (Matryoshka 1536d $\to$ 128d, ColPali patch matrices, CLIP 512d).
   * Validates schema integrity and cryptographic tenant tagging.
5. **PURGE STALE (Version Lifecycle):**
   * Looks up existing documents with identical `document_id`.
   * Flags older versions as `is_active_version = false` with a `soft_deleted_at` timestamp.
6. **LOAD (Qdrant Point Upsert):**
   * Writes new vector records and metadata payloads to Qdrant with `is_active_version = true`.
7. **CACHE INVALIDATION:**
   * Emits an asynchronous cache invalidation event:
     `REDIS.UNLINK(SCAN "cache:*:DOC_ID:*")`
   * Ensures stale cached answers derived from outdated document versions are instantly purged.
8. **STATUS COMPLETION:**
   * Updates `task_id` status to `SUCCESS` in Redis; emits an audit log to OpenTelemetry.

---

## 3. Soft-Delete Versioning & Lifecycle Strategy

Immediate physical deletion of vector points during updates causes severe index degradation, locks HNSW graphs, and makes rollbacks impossible if an erroneous document was ingested. A.X.I.O.M. enforces a **3-State Version Lifecycle**:

```text
v1.pdf (ACTIVE)
  │
  │ Admin uploads v2.pdf
  ▼
v1.pdf: is_active_version = false, soft_deleted_at = '2026-07-30T14:00:00Z'
v2.pdf: is_active_version = true  ← Live user queries immediately route here
  │
  │ 30-Day Retention Window (Instant Rollback Possible)
  ▼
[Day 31] Background CRON Hard-Purge: Permanently cleans unreferenced vectors from Qdrant
```

---

## 4. Decentralized Department Retention Policies

Different business units operate under fundamentally different regulatory requirements. Rather than forcing a rigid global retention policy, A.X.I.O.M. applies metadata-driven retention rules:

* **Legal & Regulatory:** `retention_policy = "PERMANENT_IMMUTABLE"`. Soft-delete is disabled; all versions are preserved indefinitely for compliance discovery.
* **Human Resources:** `retention_policy = "7_YEAR_STATUTORY"`. Automated hard-purge occurs 7 years after the employee separation date.
* **Engineering & Product:** `retention_policy = "ROLLING_3_VERSIONS"`. Older revisions beyond the last 3 versions are hard-purged after 14 days to prevent stale API documentation drift.
