## Evaluation Results & System Benchmarks

The system was evaluated offline across ~1,000 job postings using 5 domain-specific test queries. We compared standalone lexical search (**BM25**), semantic vector search (**Dense** via `BAAI/bge-small-en-v1.5`), and **Hybrid Search** (Reciprocal Rank Fusion, $k=60$).

### Performance Metrics ($k=10$)

| Search Query | Target Docs | BM25 P@10 | BM25 R@10 | Dense P@10 | Dense R@10 | Hybrid P@10 | Hybrid R@10 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Data Scientist Python Machine Learning | 50 | **0.90** | **0.18** | 0.80 | 0.16 | 0.80 | 0.16 |
| Remote Backend Java Developer Spring | 22 | **1.00** | **0.45** | 0.70 | 0.32 | 0.80 | 0.36 |
| DevOps Engineer AWS Kubernetes Docker | 150 | **1.00** | **0.07** | **1.00** | **0.07** | **1.00** | **0.07** |
| Frontend React TypeScript Web Developer | 110 | **1.00** | **0.09** | 0.90 | 0.08 | **1.00** | **0.09** |
| German speaking Project Manager | 617 | **1.00** | **0.02** | 0.90 | 0.01 | **1.00** | **0.02** |

---

### Retrieval Trade-Off Analysis

* **BM25 (Lexical):** Delivers the highest overall precision at $k=10$ ($0.90–1.00$) by enforcing exact term matches. However, it completely misses relevant job postings that use synonyms or alternate terminology when exact keywords are absent.
* **Dense Retriever (`bge-small-en-v1.5`):** Captures conceptual similarity and context without requiring exact keyword overlaps. Top-10 precision drops slightly ($0.70–1.00$) on keyword-dense tech-stack queries due to broader semantic matching.
* **Hybrid Search (RRF, $k=60$):** Combines the exact keyword precision of BM25 with the conceptual reach of Dense embeddings. It restores precision drops observed in standalone Dense retrieval (e.g., boosting Java Developer P@10 from $0.70$ back to $0.80$) while maintaining stable recall across query structures.

---

## Technical Limitations & Constraints

### 1. Memory (RAM) Infrastructure Bottleneck
* **Render Free-Tier Cap (512 MB RAM):** Running Dense vector models in production caused instant Out-Of-Memory (OOM) worker crashes (`Exited with status 1`). 
* **ONNX Runtime Allocation Overhead:** Switching from PyTorch to ONNX Runtime (`fastembed`) eliminated heavy Python framework dependencies, but ONNX still allocates ~1.5 GB of RAM upon graph initialization. 
* **Deployment Scope:** As a result, `dense.py` and `hybrid.py` are disabled in the live Render web deployment and replaced by an in-memory, zero-allocation BM25 pipeline. Dense/Hybrid search remains fully functional in local offline environments.

### 2. Ingestion Scope & Data Pipeline Bias
* **Two API Endpoints Only:** Data collection is restricted to 2 job board endpoints, limiting corpus volume to ~1,000 processed records.
* **Specialized Skill Bias:** The ingested dataset is heavily skewed toward specialized software engineering and tech roles, making the current index unrepresentative of the broader general job market.
* **Translation Rate Limits:** Using `deep-translator` unauthenticated web endpoints eliminates local translation RAM overhead, but requires strict rate-limiting (0.2s request pauses with exponential backoff) during bulk corpus ingestion to avoid HTTP 429 IP blocks.

### 3. Evaluation Methodology Limitations
* **Synthetic Regex Ground Truth:** Ground truth relevance was generated programmatically using regex constraints (`must_have` / `nice_to_have`). This introduces an inherent bias toward exact string matches, artificially inflating BM25 precision scores while penalizing Dense retrieval when it correctly returns semantically relevant synonyms (e.g., *"AI Engineer"* instead of *"Python"*).
* **Low Recall on Broad Queries:** Fixed-window evaluation ($k=10$) against high-volume broad queries yields low overall recall (e.g., searching "Project Manager" across 617 matching documents caps maximum achievable $R@10$ at $0.016$).