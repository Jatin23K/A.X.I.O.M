# A.X.I.O.M. — Module 04: Enterprise Security & 2-Checkpoint RBAC
## Zero-Trust Architecture · JWT Gateway · Vector Pre-Filtering · Cryptographic Isolation

> **Elevator Summary:** A.X.I.O.M. enforces an uncompromising Zero-Trust security model across two independent, decoupled checkpoints. Checkpoint 1 (API Gateway) validates JWT bearer tokens in under 0.5ms and blocks unauthorized route access before any backend service or cache is touched. Checkpoint 2 (Qdrant HNSW Pre-Filtering) enforces document-level and chunk-level multi-tenant security by filtering vector traversal exclusively to records matching the caller's tenant, clearance tier, and department — before any similarity calculations occur. The cache layer is cryptographically isolated via Role-Scoped Cache Keys, guaranteeing zero cross-tenant vector leakage.

---

## 1. The 2-Checkpoint Security Architecture

```text
HTTP REQUEST (Bearer JWT Token)
  │
  ▼
[ CHECKPOINT 1: API Gateway (< 0.5ms) ]
  │  • Cryptographically verifies RS256 JWT signature & expiration
  │  • Extracts: tenant_id, department_id, clearance_level, user_roles
  │  • Computes: Permission_Hash = SHA256(tenant + dept + clearance)
  │  • IF INVALID ──► HTTP 401 Unauthorized (Request drops immediately!)
  │
  ▼ (Checkpoint 1 Passed)
[ Cache Check (Role-Scoped Key) ] ──► (Cache Miss)
  │
  ▼
[ CHECKPOINT 2: Qdrant HNSW Pre-Filtering (< 20ms) ]
  │  • Enforces document metadata boolean bitmask BEFORE vector traversal:
  │    Filter: tenant_id == user_tenant
  │       AND department_id == user_dept
  │       AND required_clearance <= user_clearance
  │       AND is_active_version == true
  │  • Vectors failing the filter are completely invisible to the search index!
  │
  ▼ (Checkpoint 2 Passed)
[ Similarity Search (HNSW / ColPali MaxSim) ] ──► Authorized Context Only
```

---

## 2. Why Pre-Filtering is Mathematically Required (vs. Post-Filtering)

Traditional vector search implementations frequently make the catastrophic design error of **Post-Filtering**:
1. Execute unconstrained $K$-nearest neighbor search across the entire multi-tenant collection ($K=10$).
2. Inspect the retrieved results and drop chunks where the user lacks read permissions.

### The Two Fatal Failures of Post-Filtering
1. **Candidate Starvation:** If an unprivileged user asks a query related to a company-wide initiative, all Top-10 nearest chunks in vector space may belong to executive confidential memos. Dropping unauthorized chunks leaves the user with $0$ or $1$ candidate, causing retrieval starvation even when public internal documents exist.
2. **Side-Channel Information Leakage:** In multi-tenant systems, post-filtering leaks vector presence and similarity metadata through timing variations and cache footprints.

### A.X.I.O.M.'s Solution: HNSW Graph Pre-Filtering
Qdrant builds payload indexes directly over metadata fields. Before traversing the graph edges, Qdrant applies an initial bitmask filter. Non-permitted vectors are **treated as non-existent**, guaranteeing:
* The user always retrieves a full set of $K$ relevant chunks from their authorized subset.
* Zero unauthorized embeddings are ever evaluated, touched, or cached.

---

## 3. The 4-Tier Enterprise Clearance Matrix

A.X.I.O.M. structures corporate knowledge into four discrete clearance levels:

| Clearance Level | Classification | Example Artifacts | Authorized Personnel |
| :---: | :--- | :--- | :--- |
| **Level 0** | **Public** | Marketing collateral, public API docs, press releases | All authenticated employees + external clients |
| **Level 1** | **Internal** | Engineering RFCs, standard operating procedures, HR guidelines | Full-time employees |
| **Level 2** | **Confidential** | Quarterly department revenue, salary bands, product roadmaps | Department managers & senior engineers |
| **Level 3** | **Restricted** | Board minutes, M&A filings, audit logs, root credentials | C-Suite executives & designated compliance officers |

---

## 4. Role-Scoped Cache Key Isolation

To ensure cached responses inherit the same Zero-Trust guarantees as vector search, every cache entry is derived via cryptographic hashing:

$$\text{CacheKey} = \text{SHA256}\Big( \text{TenantID} \parallel \text{DeptID} \parallel \text{ClearanceLevel} \parallel \text{NormalizedQuery} \Big)$$

* **Cross-Tenant Barrier:** Tenant A and Tenant B asking the exact same question produce completely disjoint cache keys.
* **Privilege Escalation Immunity:** A Level 1 employee asking the exact same question as a Level 3 executive cannot hit the Level 3 cached answer.
