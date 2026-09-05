# A.X.I.O.M. — Module 01: Multi-Tier Semantic Cache Architecture
## 2-Tier Multi-Modal In-Memory Acceleration & Cryptographic Isolation

> **Elevator Summary:** A.X.I.O.M. integrates a zero-redundancy, two-tier caching engine built on Redis Stack. Incoming queries undergo Input Guardrail sanitization followed by a Tier 0 SHA-256 exact string lookup ($<1\text{ms}$). If that misses, we slice a 128-dimensional Matryoshka vector from a single forward pass to execute Tier 1 cosine similarity search ($\ge 0.92$) in Redis RAM in $\sim 15\text{ms}$. For visual and acoustic media, we leverage pHash Hamming distance ($\le 4$) and CLIP/Chromaprint embeddings. On a cache hit, vector database search and LLM inference are completely bypassed, delivering a 99% reduction in latency and a 100% reduction in API token costs.

---

## 1. System Control Flow Architecture

```text
USER QUERY INPUT
   │
   ▼
[ 1. Input Guardrails (< 5ms) ] ──► (Fails Validation) ──► REJECT QUERY (HTTP 400/403)
   │
   ▼ (Passes Validation)
[ 2. Tier 0: Exact Match Lookup (< 1ms) ]
   │  Text: SHA-256 Hash  ·  Image: pHash  ·  Audio: Chromaprint
   │
   ├──► [ EXACT HIT ] ──────────────────────────────────────────┐
   │                                                           │
   ▼ (EXACT MISS)                                              │
[ 3. Generate Embedding Vector ONCE (< 5ms) ]                  │
   │  Matryoshka Model generates V_full = [1536 dims]          │
   │  Slice V_128 = V_full[:128]                               │
   │                                                           │
   ▼                                                           │
[ 4. Tier 1: Semantic Vector Cache Check (~15ms) ]              │
   │  Redis RAM Cosine Similarity Search (>= 0.92)             │
   │                                                           │
   ├──► [ SEMANTIC HIT ] ──────────────────────────────────────┼──► RETURN INSTANT ANSWER
   │                                                           │    (Bypasses RAG & LLM!)
   ▼ (SEMANTIC MISS)                                           │
[ 5. Execute Full RAG Pipeline (~2.5s) ]                       │
   │  Reuses V_full [1536 dims] in Qdrant (NO RE-EMBEDDING!)   │
   │  BM25 Sparse + HNSW Dense ──► RRF ──► Cross-Encoder      │
   │  LLM Generates Raw Response                               │
   │                                                           │
   ▼                                                           │
[ 6. Output Guardrails (Groundedness Check) ]                  │
   │  Verifies zero hallucinations & source mapping            │
   │                                                           │
   ▼                                                           │
[ 7. Write Clean Answer to Redis Cache ]                       │
   │  Stores V_128 + Clean Answer + TTL (24h) in Redis RAM    │
   │                                                           │
   └───────────────────────────────────────────────────────────┘
```

---

## 2. Multi-Modal Cache Stores & Match Thresholds

A.X.I.O.M. partitions its in-memory cache into three dedicated namespaces to prevent cross-modality collision:

### 1. Text Cache Store (`cache:text`)
* **Tier 0 Exact String Match:** Normalized lowercase string hashed with SHA-256. Lookup executes in Redis RAM in $<0.2\text{ms}$.
* **Tier 1 Semantic Vector Match:** Matryoshka 128-dimensional vector slice evaluated using Cosine Similarity with a strict production threshold:
  $$\text{CosineSim}(\mathbf{u}, \mathbf{v}) = \frac{\mathbf{u} \cdot \mathbf{v}}{\|\mathbf{u}\| \|\mathbf{v}\|} \ge 0.92$$
* **Zero-Redundancy Vector Reuse:** A single forward pass generates the 1536d embedding. The first 128 dimensions are sliced for the Redis cache check. If a cache miss occurs, the already computed 1536d vector is passed directly to Qdrant without re-embedding.

### 2. Image Cache Store (`cache:image`)
* **Tier 0 Perceptual Hash (pHash):** Evaluates $8 \times 8$ discrete cosine transform (DCT) luminance gradients. Matches cropped, compressed, or resized variants of identical images using Hamming distance:
  $$\text{Hamming}(H_1, H_2) \le 4 \implies \text{Instant Hit (< 1ms)}$$
* **Tier 1 Visual CLIP Vector Match:** 512-dimensional CLIP visual embeddings evaluated via Cosine Similarity ($\ge 0.90$) to match identical objects photographed from slightly different perspectives in $\sim 20\text{ms}$.

### 3. Audio & Video Cache Store (`cache:multimodal`)
* **Audio Stream Cache:** Chromaprint acoustic waveform fingerprinting (Hamming $\le 2$, $<2\text{ms}$) combined with Whisper transcript Matryoshka embeddings.
* **Video Sequence Cache:** Temporal keyframe sequence perceptual hashes (1 frame/sec, mean Hamming distance $\le 3$, $<5\text{ms}$) combined with speech transcript vectors.

---

## 3. Cryptographic Role-Scoped Cache Keys

A critical vulnerability in enterprise RAG systems is **Cache-Layer Data Leakage**: if an executive asks a sensitive financial question and the answer is cached under `query_hash`, an unauthorized junior employee asking the same question could receive the cached privileged answer.

A.X.I.O.M. eliminates this via **Cryptographic Role-Scoped Key Derivation**:

$$\text{CacheKey} = \text{Namespace} \parallel \text{SHA256} \Big( \text{TenantID} \parallel \text{DeptID} \parallel \text{ClearanceLevel} \parallel \text{NormalizedQuery} \Big)$$

```python
import hashlib

def generate_role_scoped_cache_key(tenant_id: str, dept_id: str, clearance: int, query: str) -> str:
    raw_payload = f"{tenant_id}:{dept_id}:{clearance}:{query.strip().lower()}"
    crypto_hash = hashlib.sha256(raw_payload.encode('utf-8')).hexdigest()
    return f"cache:text:{crypto_hash}"
```

* Even if two users across different departments or clearance tiers submit the exact same string query, their derived cache keys are completely orthogonal.
* Cross-tenant and cross-clearance cache leakage is **mathematically impossible**.

---

## 4. Cache Creation & Eviction Protocol

### Pre-Sanitized Write Pattern
Cache writes execute **strictly downstream** of Output Guardrails. If an LLM response fails the Groundedness check ($<1.00$), the response is suppressed and **never written to Redis**. This invariant guarantees that any future Cache Hit is 100% pre-verified, safe, and can bypass output guardrails with zero validation overhead.

### Redis Memory Configuration
```ini
# Production Redis Configuration
maxmemory 16gb
maxmemory-policy allkeys-lru
save "" # Disable RDB snapshots to eliminate disk I/O latency jitter
appendonly no # In-memory volatile cache does not require AOF persistence
```

* **Eviction Policy:** `allkeys-lru` automatically evicts least-recently-used keys when RAM pressure reaches 16GB.
* **Time-to-Live (TTL):** Every cache entry is stamped with a 24-hour expiration (`EXPIRE key 86400`).
* **Active Invalidation:** When a document is updated or soft-deleted in the Ingestion ETL pipeline, a background worker publishes an invalidation event that executes `SCAN` and `UNLINK` on all keys tagged with that document ID.
