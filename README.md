# Job Information Retrieval System

A prototype semantic search and CV matching engine for cross-lingual (English/German) job market analysis. Built for rapid deployment on free-tier platforms with lexical (BM25) retrieval as the primary ranking mechanism.

**Live Demo:** [Render URL — add after deployment]

---

## Key Features

* **Lexical Search (BM25):** Term-matching retrieval with query preprocessing (lemmatization, stopword removal) aligned to corpus indexing. Includes relative percentage normalization to convert raw BM25 scores into intuitive 0–100% relevance matches.
* **Multi-Format Resume Parser:** Supports `.pdf`, `.docx`, and `.txt` uploads via `pdfplumber` and `python-docx`.
* **Cross-Lingual Pipeline:** German job listings are translated to English at ingestion time via `deep-translator` (API wrapper with zero-RAM overhead).
* **Metadata Filtering:** Hard-constraint filters for **Seniority Level** (Entry, Mid, Senior) and **German Language Requirement**.
* **Production WSGI Server:** Gunicorn with single-worker configuration optimized for 512 MB RAM constraints.

---

## Project Structure

```text
job-ir-system/
├── app.py                     # Main Flask web application (BM25-only deployment)
├── requirements.txt           # Python dependency manifests
├── README.md                  # Project documentation
├── data/
│   ├── processed/
│   │   └── jobs.parquet       # Preprocessed, tokenized, and indexed corpus (~1K postings)
│   └── uploads/               # Temporary directory for uploaded CV files
├── eval/
│   └── evaluate.py            # IR evaluation metrics script (P@10, R@10)
├── src/
│   ├── cv_extractor.py        # Unified parser for PDF, DOCX, and TXT files
│   ├── ingestion.py           # Scraping and raw data ingestion pipeline
│   ├── preprocessing.py       # NLP tokenization, lemmatization, and API translation
│   └── retrieval/
│       ├── bm25.py            # Lexical sparse retriever implementation
│       ├── dense.py           # Dense vector retriever (implemented, disabled in deployment)
│       └── hybrid.py          # Reciprocal Rank Fusion algorithm (implemented, disabled in deployment)
├── static/                    # Static assets (CSS, JS) — used for localhost testing
└── templates/
    └── index.html             # Responsive web UI with dynamic filters — used for localhost testing
```

**Note on UI:** The `templates/index.html` and `static/` directory provide a full web interface for local development and testing. For deployment on Render or other platforms, the same Flask template rendering is used.

---

## Evaluation Results

Evaluation conducted on 5 representative queries against a corpus of ~1,000 job postings. Metrics measured at cutoff k=10.

| Query | Relevant Docs | BM25 P@10 | BM25 R@10 |
|---|---|---|---|
| Data Scientist Python Machine Learning | 50 | 0.90 | 0.18 |
| Remote Backend Java Developer Spring | 22 | 1.00 | 0.45 |
| DevOps Engineer AWS Kubernetes Docker | 150 | 1.00 | 0.07 |
| Frontend React TypeScript Web Developer | 110 | 1.00 | 0.09 |
| German speaking Project Manager | 617 | 1.00 | 0.02 |

**Query definitions for reproducibility:**
```python
queries = [
    {"query": "Data Scientist Python Machine Learning", "must_have": ["python"], "nice_to_have": ["data", "scientist", "machine", "learning", "ai", "ml"]},
    {"query": "Remote Backend Java Developer Spring", "must_have": ["java"], "nice_to_have": ["spring", "backend", "developer", "engineer", "remote"]},
    {"query": "DevOps Engineer AWS Kubernetes Docker", "must_have": [], "nice_to_have": ["devops", "kubernetes", "k8s", "aws", "docker", "cloud", "terraform"]},
    {"query": "Frontend React TypeScript Web Developer", "must_have": [], "nice_to_have": ["react", "typescript", "javascript", "frontend", "vue", "angular", "web"]},
    {"query": "German speaking Project Manager", "must_have": [], "nice_to_have": ["project", "manager", "management", "agile", "scrum", "product", "german", "deutsch"]}
]
```

### Analysis

| Metric | Observation |
|---|---|
| **Precision@10** | Excellent (0.90–1.00). BM25 reliably surfaces relevant documents when keyword overlap exists. |
| **Recall@10** | Poor (0.02–0.45). Top-10 cutoff is insufficient for broad queries with 100–600+ relevant documents. Recall degrades as query generality increases. |

**Key insight:** BM25 excels at precision-optimized ranking but is fundamentally limited by the fixed result window. This is expected behavior for lexical retrieval on small cutoffs, not a system defect.

---

## Known Limitations

### 1. Memory (RAM) Constraints & Dense Retrieval Exclusion
* **Initial Embedding Attempt:** We initially attempted to implement semantic search using sentence transformers and dense vector retrieval.
* **Model Scaling Failure:** During deployment on Render's free tier (512 MB RAM limit), loading the models triggered immediate Out-Of-Memory (OOM) crashes (`Exited with status 1`). We attempted switching to smaller embedding models to reduce the footprint, but this failed to resolve the crashes. The core issue stems from ONNX Runtime (`fastembed`), which allocates ~1.5 GB of RAM at initialization regardless of the underlying model size.
* **Prototype Scope:** Consequently, `dense.py` and hybrid Reciprocal Rank Fusion (RRF) were excluded from the deployed prototype.
* **Local Evaluation Validated:** Despite deployment exclusion, dense retrieval was successfully evaluated offline in a local hosting environment using FastEmbed's default model (`bge-small-en-v1.5`), confirming proper execution with no invalid model string errors.

### 2. Language Translation Overhead & API Switch
* **Heavy Dependency Removal:** The ingestion pipeline originally utilized `argostranslate` for offline German-to-English translation. However, `argostranslate` required Stanza and PyTorch as core frameworks, adding ~2.5 GB of CUDA/GPU binaries during build and consuming 600 MB to 1.5 GB+ of RAM at runtime. This caused instant startup crashes on 512 MB instances.
* **Lightweight Migration:** We switched the translation engine from `argostranslate` to `deep-translator`. By routing text to external public web translation endpoints (such as Google Translate), local RAM overhead dropped from ~1 GB to under 10 MB, enabling seamless deployment on zero-budget tiers.
* **Ingestion Rate Limits:** Because free web translation endpoints (via `deep-translator`) rely on HTTP requests without authentication, bulk data ingestion is subject to strict rate limits. Ingestion scripts require small request pauses and exponential backoff to prevent temporary IP blocks (HTTP 429).

### 3. Data Ingestion Scope & Job Bias
* **Endpoint Limitation:** The data ingestion pipeline collected postings from only 2 ingestion endpoints.
* **Skill-Level Skew:** As a result of the selected endpoints, the indexed job corpus is heavily leaned toward skilled, specialized, and technical professions, which may not accurately reflect the broader distribution of the general labor market.

### 4. Low Recall on Broad Queries
* **Fixed Result Window:** Lexical search is restricted by a fixed top-10 result cutoff against large sets of relevant documents.
* **High Document Volume:** For generic queries (e.g., "German speaking Project Manager" with 617 relevant documents), users see only a tiny fraction of matching jobs (max possible R@10 of 0.016).
* **Mitigation:** Future iterations should implement pagination or "load more" functionality to increase the effective cutoff window.

### 5. Cold Starts on Free Tier
* **Instance Spin-Down:** Render's free tier automatically spins down web services after 15 minutes of inactivity.
* **Startup Latency:** The initial request following an idle period incurs a ~5–10 second delay while Python initializes, NLTK datasets load, and the BM25 sparse index is built in memory.
* **Mitigation:** Upgrading to a paid host tier provides always-on instances and eliminates boot latency.

---

## Setup & Deployment

### Local Development
```bash
git clone https://github.com/yourusername/job-ir-system.git
cd job-ir-system
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python app.py
```
Visit `http://127.0.0.1:5000` — the `templates/index.html` UI will be served by Flask.

### Render (Free Tier)
* **Step 1:** Fork and push this repository to GitHub.
* **Step 2:** Connect the repository at [render.com](https://render.com).
* **Step 3:** Create a new Web Service using Python 3.
* **Step 4:** Set the Build Command to `pip install -r requirements.txt`.
* **Step 5:** Set the Start Command to `gunicorn app:app`.
* **Step 6:** Select the `Free` instance plan.

The `templates/` and `static/` directories are automatically served by Flask in production.

---

## Re-enabling Hybrid Search

Dense retrieval and RRF fusion are fully implemented in the codebase and ready for production on high-memory servers. To enable:

* **Step 1:** Uncomment `fastembed>=0.2.0` in `requirements.txt`.
* **Step 2:** Uncomment the dense module export in `src/retrieval/__init__.py`:
  ```python
  from .dense import DenseRetriever
  __all__ = ["BM25", "DenseRetriever", "reciprocal_rank_fusion"]
  ```
* **Step 3:** Uncomment the dense indexing and hybrid RRF scoring logic in `app.py`:
  ```python
  from retrieval import DenseRetriever, reciprocal_rank_fusion
  dense_retriever = DenseRetriever()
  dense_retriever.fit_corpus(dense_corpus_texts, batch_size=64)
  # Use hybrid ranking instead of BM25-only
  hybrid_ranking, rrf_scores = reciprocal_rank_fusion([bm25_ranking, dense_ranking], k=60)
  ```
* **Step 4:** Deploy the application to a paid hosting tier with at least 2–4 GB of RAM (e.g., Render Starter, Cloud Run, or Hetzner CX21).

---

## Tech Stack

| Component | Technology |
|---|---|
| **Web Framework** | Flask + Gunicorn |
| **UI** | Jinja2 templates (`templates/`), static assets (`static/`) |
| **Data Processing** | pandas, pyarrow |
| **NLP** | NLTK (lemmatization, stopwords) |
| **Translation** | deep-translator (Google Translate API wrapper) |
| **Document Parsing** | pdfplumber, python-docx |
| **Retrieval** | Custom BM25 implementation with relative score normalization |
| **Dense Embeddings** | fastembed / ONNX Runtime (disabled in deployment) |
| **Deployment** | Render (free tier, 512 MB RAM limit) |

---

## License

MIT