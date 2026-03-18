# AI Legislative Analyzer — Citizen's Dashboard

> **Project 3** · Token Compression · Information Density · Indian Law

A full-stack application that takes dense Indian parliamentary bills and legal documents and transforms them into plain-language citizen summaries. The core technique is **Token Compression** — extractively filtering and deduplicating text before it reaches the generative model, maximizing information density (value per token).

---

## Project Structure

```
CREATE_BACKEND/
├── app/
│   ├── models/
│   │   └── hf_models.py          # HuggingFace model loader (lazy, cached)
│   ├── routes/
│   │   ├── summarize.py          # POST /api/summarize/text  &  /pdf-base64
│   │   └── upload.py             # POST /api/upload
│   ├── services/
│   │   ├── chunking.py           # Sentence-level chunking with stride overlap
│   │   ├── compression.py        # ★ Token Compression (TF-IDF + dedup + boilerplate strip)
│   │   ├── pipeline.py           # Orchestrator: parse → compress → chunk → summarise
│   │   └── summarizer.py         # HF pipeline wrapper + hierarchical re-summarisation
│   └── utils/
│       ├── pdf_parser.py         # pdfplumber extraction + header/footer removal
│       └── text_cleaner.py       # Unicode normalisation, artefact removal
├── main.py                       # FastAPI app, CORS, router registration
├── requirements.txt
└── frontend/
    └── LegislativeAnalyzer.jsx   # React dashboard
```

---

## Quickstart

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Run the backend

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

The first request will download the BART model (~1.6 GB). Subsequent requests use the cached model.

### 3. Run the frontend

Copy `LegislativeAnalyzer.jsx` into your React project (Vite or CRA):

```bash
# Vite
npm create vite@latest citizen-dashboard -- --template react
cd citizen-dashboard
cp LegislativeAnalyzer.jsx src/App.jsx
npm install
npm run dev
```

---

## API Reference

### `POST /api/upload`
Upload a PDF or `.txt` file.

**Request**: `multipart/form-data` with field `file`

**Response**:
```json
{
  "filename": "draft_bill.pdf",
  "size_mb": 1.2,
  "word_count": 18400,
  "char_count": 112000,
  "text_preview": "...",
  "full_text": "..."
}
```

---

### `POST /api/summarize/text`
Summarise pre-extracted plain text.

**Request**:
```json
{
  "text": "Full document text here...",
  "keep_ratio": 0.45
}
```

**Response**:
```json
{
  "summary": "The bill proposes...",
  "key_points": ["Point 1...", "Point 2..."],
  "word_count": 18400,
  "stats": {
    "original_tokens": 25000,
    "compressed_tokens": 8200,
    "token_reduction_pct": 67.2,
    "char_reduction_pct": 61.8,
    "information_density_score": 0.328
  }
}
```

---

### `POST /api/summarize/pdf-base64`
Summarise a base64-encoded PDF directly.

**Request**:
```json
{
  "pdf_base64": "<base64 string>",
  "keep_ratio": 0.45
}
```

---

## Token Compression Pipeline

This is the core innovation addressing the problem constraint: **documents > 100k tokens**.

```
Raw Document
     │
     ▼
[1] Boilerplate Strip        ← removes "Be it enacted", "Gazette" notices, etc.
     │
     ▼
[2] Deduplication            ← cosine similarity; removes near-identical sentences
     │
     ▼
[3] TF-IDF Extraction        ← keeps top keep_ratio% most informative sentences
     │
     ▼
[4] Chunking (900 tok each)  ← sentence-safe, stride overlap for context
     │
     ▼
[5] Per-chunk Summarisation  ← BART generates chunk summaries
     │
     ▼
[6] Hierarchical Merge       ← if >1 chunk, combined summaries re-summarised
     │
     ▼
Final Summary + Key Points
```

**Information Density Score** = `compressed_tokens / original_tokens`  
A score of 0.33 means 33% of the original tokens delivered the same core information.

---

## Configuration

### Swap the AI Model

Edit `app/models/hf_models.py`:

```python
# Current: fast, good quality
MODEL_NAME = "facebook/bart-large-cnn"

# For very long documents (16k token context):
MODEL_NAME = "allenai/led-base-16384"

# For Google's long-T5:
MODEL_NAME = "google/long-t5-tglobal-base"
```

### Tune Compression Aggressiveness

The `keep_ratio` parameter (default `0.45`) controls what fraction of sentences survive extractive compression:

| `keep_ratio` | Tokens saved (approx) | Use case |
|---|---|---|
| `0.10` – `0.25` | 75–90% | Very large documents, speed priority |
| `0.35` – `0.50` | 50–65% | **Recommended** — good balance |
| `0.60` – `0.90` | 10–40% | High-detail summaries, short docs |

---

## Key Design Decisions

| Decision | Rationale |
|---|---|
| Extractive pre-compression | Cuts tokens *before* the LLM sees them — saves energy and cost |
| TF-IDF over embeddings | No GPU needed for compression; deterministic, interpretable |
| Sentence-level chunking | Never cuts mid-sentence; stride overlap keeps cross-chunk context |
| Hierarchical summarisation | Handles 100k+ token documents without truncation |
| Legal boilerplate regex | Indian legislative text has predictable formulaic sections — strip them early |

---

## Requirements

- Python 3.10+
- 8 GB RAM (16 GB recommended for BART-large)
- GPU optional but speeds up inference 5-10x

---

## Judging Criteria: Information Density

The `stats.information_density_score` in every API response directly measures value-per-token. Lower = more efficient. The system targets a score below `0.40` on typical Indian legislative documents (60%+ token reduction).# AI-Legislative-Analyzer
