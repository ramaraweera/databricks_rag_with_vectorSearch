# Databricks RAG with Vector Search

> **Author:** Ravi Amaraweera — Senior Data Architect / Analytics Engineer  
> **Stack:** Databricks Vector Search · Foundation Model APIs · LangChain  
> **Data:** Apache Airflow open-source documentation (Apache 2.0 licence)  
> **Estimated cost to run:** \$5–\$15 within Databricks' free \$400 trial credits

---

## Overview

This project demonstrates a **production-grade Retrieval-Augmented Generation (RAG)** pipeline built entirely on the Databricks Lakehouse platform — without relying on the Mosaic AI Agent Framework. It is designed as a reference implementation for data engineers and architects who want to understand how each component of a RAG system is assembled from first principles.

The pipeline answers natural-language questions about Apache Airflow documentation by:

1. Scraping and chunking the Airflow docs
2. Storing chunks as a **Delta Table** in Unity Catalog
3. Vectorising them using Databricks' **managed embedding model** (`databricks-gte-large-en`)
4. Serving a **Databricks Vector Search** index via a dedicated compute endpoint
5. Wiring a **LangChain retrieval chain** to query the index and synthesise answers with `llama-4-maverick` via the Foundation Model API

```
User Question
      │
      ▼
 [LangChain Chain]
      │
      ├──► Vector Search (semantic retrieval, top-k chunks)
      │         │
      │         └── Delta Table: rag_portfolio.doc_search.airflow_docs_chunks
      │
      └──► Foundation Model API (llama-4-maverick)
                │
                └── Grounded answer returned to user
```

---

## Repository Structure

```
databricks_rag_with_vectorSearch/
├── README.md                          ← You are here
├── GETTING_STARTED.md                 ← Beginner's guide to every step
└── databricks_rag_vector_search.ipynb ← Main notebook (run top-to-bottom)
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| Databricks workspace | Premium or above (Unity Catalog must be enabled) |
| DBR version | Databricks Runtime 14.3 LTS or later (Python 3.10+) |
| Cluster type | Single-node is sufficient; 8–16 GB RAM recommended |
| Permissions | `CREATE CATALOG`, `CREATE SCHEMA`, Vector Search endpoint creation |
| Network | Outbound internet access (to scrape Apache Airflow docs) |

No external API keys are required. All models are served via **Databricks Foundation Model APIs**, which authenticate automatically using your workspace token.

---

## Quickstart

### 1. Clone the repository

```bash
git clone https://github.com/ramaraweera/databricks_rag_with_vectorSearch.git
```

### 2. Import the notebook into Databricks

- Open your Databricks workspace
- Navigate to **Workspace → Import**
- Upload `databricks_rag_vector_search.ipynb`

### 3. Attach a cluster

Attach the notebook to a cluster running **DBR 14.3 LTS** or later.

### 4. Run cells sequentially

Run each cell from top to bottom. Key milestones:

| Cell | Action | Wait time |
|---|---|---|
| 1 | Create Unity Catalog + schema | < 1 min |
| 2 | Install pip dependencies (kernel restarts) | 2–3 min |
| 3 | Scrape Airflow docs | < 1 min |
| 4 | Chunk documents | < 1 min |
| 5–6 | Write to Delta Table | < 1 min |
| 7 | Enable Change Data Feed | < 1 min |
| 8 | Create Vector Search endpoint | **5–10 min** |
| 9 | Wait for endpoint to come online | Polling |
| 10 | Create Delta Sync Index | **5–15 min** |
| 11 | Wait for index to be ready | Polling |
| 12 | Verify index row count | < 1 min |
| 13–14 | Build LangChain retriever | < 1 min |
| 15 | Run a single Q&A query | < 30 sec |
| 16 | Run batch evaluation queries | < 2 min |
| 17 | Inspect retrieved context | < 1 min |
| 18 | Save eval results to Delta | < 1 min |
| **19** | **⚠ Teardown — delete index & endpoint** | **Run this to stop billing** |

> **⚠ Cost Warning:** The Vector Search endpoint incurs charges while running. Always execute **Cell 19** at the end of your session.

---

## Tech Stack

| Component | Technology |
|---|---|
| Data platform | Databricks (Unity Catalog, Delta Lake) |
| Embeddings | `databricks-gte-large-en` via Foundation Model API |
| LLM | `llama-4-maverick` via Foundation Model API |
| Vector store | Databricks Vector Search (Delta Sync Index) |
| Orchestration | LangChain + `databricks-langchain` |
| Document ingestion | `requests` + `BeautifulSoup4` |
| Text splitting | LangChain `RecursiveCharacterTextSplitter` |
| Evaluation output | Delta Table (`rag_portfolio.doc_search.eval_results`) |

---

## Key Design Decisions

### Why no Mosaic AI Agent Framework?
This project deliberately avoids the higher-level Agent Framework to make every RAG component explicit and portable. Developers can follow exactly what happens at each step and adapt the pattern to their own data and models.

### Why Delta Sync Index?
The Delta Sync Index automatically keeps the vector index in sync as new rows are added to the Delta Table. This enables incremental ingestion without re-indexing from scratch — a critical property for production data pipelines.

### Why Apache Airflow docs?
Apache Airflow documentation is open-source (Apache 2.0), well-structured, and widely understood by data engineers — making it an ideal domain for a portfolio demonstration without licensing concerns.

---

## Customising for Your Own Data

To adapt this pipeline to your own documents:

1. **Replace Cell 3** — swap the Airflow scraper for your own data loader (`PyPDFLoader`, a SharePoint connector, an S3 reader, etc.)
2. **Update `table_name`** in Cell 5 to reflect your catalog/schema/table naming convention
3. **Adjust chunk size** in Cell 4 (`chunk_size`, `chunk_overlap`) based on your document density
4. **Update the system prompt** in Cell 13 to reflect your domain context
5. **Swap the LLM** in Cell 14 (`llama-4-maverick` → any Foundation Model API model) to match your requirements

---

## Evaluation Output

Cell 18 persists a structured evaluation table to Unity Catalog:

```
rag_portfolio.doc_search.eval_results
├── question       — the input query
├── answer         — the LLM-generated response
├── num_chunks     — number of context chunks retrieved
├── sources        — comma-separated source doc list
└── timestamp      — UTC run time
```

This table can be queried directly in Databricks SQL or visualised in a Databricks Dashboard.

---

## Teardown

**Run Cell 19 before closing your session.** It deletes:
- The Vector Search index
- The Vector Search endpoint

Failure to run this cell will result in continued endpoint charges on your Databricks account.

---

## Licence

Source code in this repository is released under the **MIT Licence**.  
Apache Airflow documentation used as demo data is licenced under the **Apache 2.0 Licence**.

---

## About the Author

**Ravi Amaraweera** is a Senior Data Architect and Analytics Engineer specialising in cloud-native data platforms, Lakehouse architectures, and applied AI/ML pipelines on Databricks and Azure.

- GitHub: [ramaraweera](https://github.com/ramaraweera)
