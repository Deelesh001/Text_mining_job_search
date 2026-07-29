# Cross-Lingual Job Search & Resume Matching System

An end-to-end Information Retrieval (IR) and recommendation engine that bridges the language gap in the German and English tech job markets. It aggregates job listings from multiple APIs, performs zero-RAM translation, and uses a hybrid retrieval pipeline (BM25 + Dense Vectors) to match user CVs against relevant postings.

## Quick Start

### Prerequisites

- Python 3.8+
- Git

### Installation

1. **Clone the repository:**

   ```bash
   git clone https://github.com/Deelesh001/Text_mining_job_search.git
   cd Text_mining_job_search
   ```

2. **Set up a virtual environment:**

   ```bash
   python3 -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

3. **Download required NLTK language data:**

   ```bash
   python3 -c "import nltk; nltk.download('stopwords'); nltk.download('wordnet'); nltk.download('omw-1.4')"
   ```

## Usage

### 1. Run the ingestion & translation pipeline

Fetch listings from Arbeitnow and Remotive APIs, detect languages, translate German postings to English, and build the local Parquet dataset:

```bash
python3 src/ingestion.py
```

### 2. Launch the web application

Start the local Flask server:

```bash
python3 app.py
```

Open your browser and navigate to `http://127.0.0.1:5000` to interact with the search interface, apply seniority/language filters, or upload a resume for automated matching.

## System architecture

- **Ingestion & caching** — Pulls job data from Arbeitnow and Remotive APIs. Uses `langdetect` to identify German (`de`) listings and routes them through a lightweight `deep-translator` API wrapper. MD5 text hashing caches translations locally, dropping API calls by over 40% and avoiding HTTP 429 rate limits.

- **Shared preprocessing pipeline** — Applies lowercasing, regex punctuation cleaning, NLTK stopword removal, and WordNet lemmatization uniformly across stored jobs, search queries, and parsed resumes to guarantee vocabulary alignment.

- **Hybrid retrieval engine**
  - *Lexical (BM25)* — Inverted index scoring with term-frequency saturation and length normalization.
  - *Dense vectors* — 384-dimensional continuous embeddings generated via `fastembed` (`bge-small-en-v1.5`), running on the ONNX Runtime for roughly 5x faster CPU inference compared to PyTorch.
  - *Reciprocal Rank Fusion (RRF)* — Combines lexical and dense rankings using `RRF = 1 / (k + rank)` with `k = 60`, balancing exact keyword precision with semantic recall.

- **Resume matching** — Parses `.pdf`, `.docx`, and `.txt` resumes using `pdfplumber` and `python-docx`. Extracted text is processed through the same query pipeline to compute similarity scores and rank job openings.

## Tech stack

| Category | Tools |
|---|---|
| Backend & web | Python 3, Flask (WSGI), Jinja2 |
| Data processing | Pandas, Apache Parquet, Requests |
| NLP & IR | NLTK, WordNet, `langdetect`, `deep-translator`, BM25 |
| Machine learning | `fastembed` (`bge-small-en-v1.5`), ONNX Runtime |
| Document parsing | `pdfplumber`, `python-docx` |

## Evaluation results & system benchmarks

The system was evaluated offline across roughly 1,000 job postings using 5 domain-specific test queries. Standalone lexical search (BM25), semantic vector search (Dense via `BAAI/bge-small-en-v1.5`), and Hybrid search (Reciprocal Rank Fusion, `k = 60`) were compared.

### Performance metrics (k = 10)

| Search query | Target docs | BM25 P@10 | BM25 R@10 | Dense P@10 | Dense R@10 | Hybrid P@10 | Hybrid R@10 |
|---|---|---|---|---|---|---|---|
| Data Scientist Python Machine Learning | 50 | 0.90 | 0.18 | 0.80 | 0.16 | 0.80 | 0.16 |
| Remote Backend Java Developer Spring | 22 | 1.00 | 0.45 | 0.70 | 0.32 | 0.80 | 0.36 |
| DevOps Engineer AWS Kubernetes Docker | 150 | 1.00 | 0.07 | 1.00 | 0.07 | 1.00 | 0.07 |
| Frontend React TypeScript Web Developer | 110 | 1.00 | 0.09 | 0.90 | 0.08 | 1.00 | 0.09 |
| German speaking Project Manager | 617 | 1.00 | 0.02 | 0.90 | 0.01 | 1.00 | 0.02 |

## Production notes & known limitations

- **Memory constraints** — Free-tier hosting environments (e.g. Render's 512 MB RAM limit) can trigger out-of-memory crashes when loading heavy local neural translation or PyTorch models. To maintain stability, live deployments run the lightweight BM25 lexical engine, while dense/hybrid retrieval requires environments with at least 2 GB RAM.
- **Evaluation bias** — The project's evaluation benchmarks rely on synthetic regex ground-truth rules, which inherently favor exact-match BM25 scoring over semantic dense retrieval.
