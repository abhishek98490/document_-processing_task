# Document Processing Task

A lightweight RAG pipeline for intelligent document processing. Given a document (PDF or image), the system classifies it, summarises it, and either extracts key dates or answers a freeform user query — all in parallel.

---


**LLM backend:** SambaNova Cloud — `Meta-Llama-3.1-8B-Instruct` via OpenAI-compatible API.

---

## Assumptions

| Dimension | Assumption |
|---|---|
| **Document classes** | Invoice, Identity Document, Contract, Medical Record, Resume, Report, Other |
| **Language** | English only |
| **Complexity** | Simple to medium (no heavily nested tables, no multi-column layouts) |
| **File size** | 2–3 pages; small data files |
| **Date formats** | Mixed (ISO, DD/MM/YYYY, Month DD YYYY, etc.) — normalised to ISO 8601 |

---

## Project Structure

```
document_-processing_task/
├── main.py                         # Entry point — orchestrates all modes
├── requirements.txt
├── .env                            # API keys (see Setup)
├── logs/                           # Auto-created, daily log files
└── src/
    ├── config.py                   # Chunk size, prompts, constants
    ├── logging.py                  # File-based logging setup
    ├── chunking.py                 # Sliding window tokeniser
    ├── vector_store.py             # ChromaDB wrapper (embed, store, retrieve)
    ├── date_normalise.py           # ISO 8601 date normalisation
    ├── data_ingestion/
    │   ├── data_loader.py          # PDF / image loader with OCR fallback
    │   └── ocr.py                  # Tesseract + OpenCV OCR engine
    └── LLM_gateway/
        └── LLM_Call.py             # SambaNova / OpenAI-compatible LLM client
```

---

## Setup

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

> **Note:** `pdf2image` requires `poppler` on the system path.
> - macOS: `brew install poppler`
> - Ubuntu: `apt install poppler-utils`
> - Windows: download from https://github.com/oschwartz10612/poppler-windows

> **Note:** `pytesseract` requires Tesseract OCR installed.
> - macOS: `brew install tesseract`
> - Ubuntu: `apt install tesseract-ocr`

### 2. Configure environment

Create a `.env` file in the project root:

```env
SAMBANOVA_API_KEY=your_api_key_here
CHROMADB_PATH=chromadb          # optional, defaults to ./chromadb
CHROMA_DISTANCE=cosine          # optional, defaults to cosine
```

Get a free SambaNova API key at: https://cloud.sambanova.ai

---

## Usage

### Date Extraction Mode (default)

Classifies the document, summarises it, and extracts all key dates.

```bash
python main.py "<path/to/document.pdf>"
```

**Output:**
```
=== Document Type ===
Identity Document

=== Summary ===
This is an Aadhaar card issued by UIDAI...

=== Expiry Date ===
None

=== Activation Date ===
2015-03-10

=== Other Dates ===
{
  "date_of_birth": "1990-07-04"
}

=== Confidence ===
0.92

=== Sources ===
adhar card.pdf (chunk 2), adhar card.pdf (chunk 4)
```

### Query Mode

Classifies and summarises the document, then answers a specific question using RAG.

```bash
python main.py "<path/to/document.pdf>" "What is the address?"
```

**Output:**
```
=== Document Type ===
Identity Document

=== Summary ===
...

=== Answer ===
The address listed on the document is...

=== Sources ===
adhar card.pdf (chunk 3)
```

### Supported File Types

| Extension | Extraction Method |
|---|---|
| `.pdf` | PyPDF2 → OCR fallback if text < 20 chars/page |
| `.jpg` / `.jpeg` | Tesseract OCR via OpenCV |
| `.png` | Tesseract OCR via OpenCV |

---

## Configuration (`src/config.py`)

| Parameter | Default | Description |
|---|---|---|
| `CHUNK_SIZE` | 200 | Tokens per chunk |
| `OVERLAP` | 30 | Overlapping tokens between chunks |
| `N_RESULTS` | 5 | Top-k chunks retrieved from ChromaDB |
| `SUMMARY_CHAR_CAP` | 2000 | Max chars sent to analyse prompt |
| `DISTANCE_METRIC` | cosine | ChromaDB similarity metric |

---

## Logging

Logs are written to `logs/MM_DD_YYYY.log`. Each run appends to the daily file. Log level: `INFO`.

---

## Known Limitations & Edge Cases

- **Ambiguous DD/MM dates:** heuristic assumes DD/MM when day ≤ 12; can misparse MM/DD American dates
- **max_tokens=150:** LLM responses are capped at 150 tokens — sufficient for structured JSON extraction but may truncate long RAG answers
- **SambaNova rate limits** — free tier may throttle under concurrent load
- **OCR quality** — low-DPI scans or handwritten text will degrade extraction accuracy

---