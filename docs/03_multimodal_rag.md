# A.X.I.O.M. — Module 03: Omnimodal RAG Engine
## ColPali Patch-Based Visual RAG · Dual-Path Audio RAG · 3-Stream Video RAG

> **Elevator Summary:** A.X.I.O.M. processes 4 distinct data modalities — Text, Image, Audio, and Video — each backed by a specialized ingestion strategy, vector schema, and retrieval algorithm. Text utilizes Matryoshka 2-vector hybrid search. Images bypass OCR entirely using ColPali Vision Transformer (ViT) spatial patch multi-vector matrices evaluated via MaxSim Late Interaction — completely eliminating the need for a Cross-Encoder reranker. Audio employs a Dual-Path pipeline combining WhisperX speech transcription with CLAP acoustic waveform fingerprinting. Video applies a 3-Stream synchronized temporal pipeline. All 4 modalities are indexed in a unified Qdrant cluster using Named Vector Fields.

---

## 1. Modality Routing Architecture

```text
INPUT QUERY
  │
  ▼
[ Modality Router ]
  │
  ├──► TEXT QUERY ──────────► Matryoshka Embedding → BM25+HNSW RRF → Cross-Encoder → LLM
  ├──► IMAGE QUERY ─────────► CLIP Query Vector → ColPali MaxSim → VLM
  ├──► AUDIO QUERY (Text) ──► WhisperX Embed → V_speech + V_acoustic → LLM
  ├──► AUDIO QUERY (Sound) ──► CLAP Embed → V_acoustic search → LLM
  └──► VIDEO QUERY ──────────► CLIP+WhisperX+CLAP → 3-Stream RRF → VLM
```

---

## 2. Image RAG — ColPali Patch-Based Multi-Vector Retrieval

Traditional OCR-based document RAG degrades severely on PDFs containing complex tables, infographics, multi-column layouts, and scientific diagrams. A.X.I.O.M. completely eliminates OCR in favor of **ColPali Visual Retrieval**:

### Building Phase (Zero-OCR Ingestion)
1. **Rasterization:** Every PDF page is rendered as a clean $1024 \times 1024$ RGB image tensor.
2. **ViT Patch Extraction:** A Vision Transformer (PaliGemma-3B / SigLIP backbone) divides each page into a $32 \times 32$ grid, producing $1,024$ distinct spatial patch embeddings per page.
3. **Multi-Vector Storage:** Instead of collapsing a page into a single averaged vector, the page is stored in Qdrant as a multi-vector matrix of $1,024$ vectors, preserving precise spatial coordinates.

### Retrieval Phase (MaxSim Late Interaction)
When a text query $Q$ arrives, it is tokenized into $N$ query token vectors $\{q_1, q_2, \dots, q_N\}$. Retrieval computes the **MaxSim Late Interaction operator** across every page $D$ with patch vectors $\{p_1, p_2, \dots, p_M\}$:

$$\text{Score}(Q, D) = \sum_{i=1}^{N} \max_{j=1}^{M} \left( \mathbf{q}_i \cdot \mathbf{p}_j^T \right)$$

* **Why No Reranker is Needed:** MaxSim evaluates token-to-patch fine-grained cross-attention natively in a single vector operation. Adding a secondary Cross-Encoder reranker would duplicate computational overhead and introduce 1.5–2.0 seconds of unnecessary latency.
* **Why No Text Normalization:** Text preprocessing (stopword removal, stemming) operates on strings. ColPali operates directly on raw pixel tensors, meaning visual elements (bold headers, table borders, colored chart bars) are intrinsically preserved.

---

## 3. Audio RAG — Dual-Path Ingestion Pipeline

Audio files contain two independent streams of information: **semantic speech** (what was spoken) and **acoustic events** (background alarms, mechanical vibrations, ambient tone). A.X.I.O.M. implements a Dual-Path pipeline:

```text
RAW AUDIO FILE (.wav / .mp3)
  │
  ├──► PATH A: SEMANTIC (SPEECH)
  │    • WhisperX STT → Word-level timestamped transcript
  │    • Text Embedding → V_speech vector
  │    • VAD Silence Chunking: 10s - 45s windows
  │
  └──► PATH B: ACOUSTIC (SOUND)
       • CLAP Mel-Spectrogram → V_acoustic 512d vector
       • Chromaprint acoustic fingerprint (Tier 0 Cache)
       • 10s sliding window chunks
```

### Unified Qdrant Multi-Vector Payload
```json
{
  "audio_id": "AUD_1042",
  "start_time": "02:15",
  "end_time": "02:45",
  "transcript": "Replace valve 4 immediately, pressure exceeding 400 PSI",
  "vectors": {
    "speech_vec": [0.88, 0.05, -0.12, "..."],
    "acoustic_vec": [0.12, -0.44, 0.91, "..."]
  }
}
```

* **Voice Activity Detection (VAD) Chunking:** Audio is chunked dynamically based on speech pauses ($>0.5\text{s}$) with a 2-second overlap, bounded between 10s minimum and 45s maximum, ensuring words are never truncated mid-syllable.

---

## 4. Video RAG — 3-Stream Synchronized Temporal Pipeline

Video represents the most computationally demanding modality. A.X.I.O.M. splits incoming video into three synchronized parallel streams:

1. **Stream 1 (Visual Keyframes):** PySceneDetect extracts keyframe images on scene transitions (adaptive content-aware thresholding, bounded at 1 frame per scene). Keyframes are embedded via ColPali / CLIP.
2. **Stream 2 (Spoken Transcript):** WhisperX extracts word-level timestamped transcripts, embedded via Matryoshka.
3. **Stream 3 (Acoustic Track):** CLAP extracts ambient sound fingerprints and non-verbal acoustic vectors.

### Temporal Synchronization & Playback Anchoring
All three vector streams share a synchronized temporal coordinate: `{"video_id": "VID_902", "timestamp_start_ms": 142000, "timestamp_end_ms": 165000}`. When a query matches, the user response includes an exact deep-link timestamp (`https://player.core.internal/watch?v=VID_902&t=142s`), allowing instant playback at the exact second the answer occurs.
