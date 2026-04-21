# Getting Started — Databricks RAG with Vector Search

> A plain-English walkthrough of every step in the notebook.  
> Written for developers who are new to Databricks and/or Retrieval-Augmented Generation (RAG).

---

## What is RAG?

**Retrieval-Augmented Generation (RAG)** is a technique that makes a Large Language Model (LLM) answer questions using *your own documents*, rather than only its training data.

Think of it as giving the LLM an open-book exam:

1. You **store** your documents in a searchable index
2. When a question arrives, you **retrieve** the most relevant passages
3. You **pass those passages to the LLM** along with the question
4. The LLM **generates a grounded answer** based on that context

This avoids hallucinations and keeps answers up-to-date without retraining the model.

```
Without RAG:  Question ──► LLM ──► Answer (may be wrong/stale)

With RAG:     Question ──► Search Index ──► Relevant chunks
                                                    │
                                         Question + Chunks ──► LLM ──► Grounded Answer
```

---

## What is Databricks?

**Databricks** is a cloud-based data and AI platform built on Apache Spark and Delta Lake. It provides:

- **Notebooks** — interactive coding environments (like Jupyter, but cloud-native)
- **Unity Catalog** — a centralised metadata and governance layer for all data assets
- **Delta Lake** — ACID-compliant table storage (think: reliable, queryable files in cloud storage)
- **Vector Search** — a managed service for storing and searching vector embeddings
- **Foundation Model APIs** — hosted LLMs and embedding models you can call like an API

You do **not** need to manage any servers or install any software locally. Everything runs in the cloud.

---

## The Big Picture

Here is how all the pieces fit together in this project:

```
[Apache Airflow Docs Website]
          │  (Cell 3 — scrape)
          ▼
[Raw text documents in Python]
          │  (Cell 4 — chunk)
          ▼
[Text chunks ~1000 chars each]
          │  (Cell 5-6 — write)
          ▼
[Delta Table: airflow_docs_chunks]
          │  (Cells 7-11 — index)
          ▼
[Vector Search Index]  ◄──── Embedding model converts text to numbers
          │
          │  (Cells 13-14 — retriever chain)
          ▼
[LangChain RAG Chain]
     │         │
     │    User Question
     │         │
     ▼         ▼
[Semantic    [LLM: llama-4-maverick]
 Search]            │
     │              ▼
     └──────► [Final Answer]
```

---

## Step-by-Step Explanation

### Cell 1 — Create the Unity Catalog and Schema

**What it does:** Creates a container (`rag_portfolio`) and a folder inside it (`doc_search`) to hold all the data objects this project needs.

**Key concepts:**
- **Catalog** → the top-level container in Unity Catalog. Think of it like a database server.
- **Schema** → a namespace inside a catalog. Think of it like a database within that server.
- `IF NOT EXISTS` → makes the cell safe to re-run without errors.

```sql
CREATE CATALOG IF NOT EXISTS rag_portfolio;
CREATE SCHEMA IF NOT EXISTS doc_search;
```

**Why it matters:** Unity Catalog governs all data access. Everything created in subsequent cells lives inside `rag_portfolio.doc_search.*`.

---

### Cell 2 — Install Dependencies

**What it does:** Installs the Python libraries needed for the pipeline:

| Library | Purpose |
|---|---|
| `langchain` | Framework for chaining LLM calls and retrievers |
| `langchain-text-splitters` | Splits long documents into smaller chunks |
| `databricks-langchain` | Databricks-specific LangChain integrations |
| `requests` | HTTP client to scrape web pages |
| `beautifulsoup4` | HTML parser to extract clean text from web pages |

> **Note:** After installation, the kernel restarts automatically. This is normal. Resume from Cell 3.

---

### Cell 3 — Scrape the Apache Airflow Documentation

**What it does:** Downloads several pages from the official Apache Airflow documentation website and extracts their plain text content.

**How it works:**
1. For each URL in the `pages` list, it makes an HTTP `GET` request
2. `BeautifulSoup` parses the HTML response
3. It finds the main content `<div>` (the article body, not the navigation or headers)
4. It extracts plain text using `.get_text()`
5. Each page becomes one document dict with `source`, `text`, and `doc_type`

**Why Apache Airflow docs?**  
They are open-source (Apache 2.0 licence), well-structured, and technically rich — ideal for demonstrating a Q&A system relevant to data engineers.

**Customisation tip:** Replace this cell with any data source — PDFs, SharePoint pages, S3 files, database records. The rest of the pipeline stays the same.

---

### Cell 4 — Chunk the Documents

**What it does:** Splits each full document into smaller overlapping pieces called **chunks**.

**Why chunking?**
- Embedding models have a token limit — you can't embed a whole document at once
- Smaller, focused chunks give better search precision
- Overlap (`chunk_overlap=150`) ensures a sentence split across a boundary is still represented

```python
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,      # each chunk is ~1000 characters
    chunk_overlap=150,    # 150-char overlap between consecutive chunks
    separators=["\n\n", "\n", ". ", " "]  # prefer splitting at natural boundaries
)
```

The `RecursiveCharacterTextSplitter` tries to split at paragraph breaks first, then line breaks, then sentence boundaries, then word boundaries — preserving semantic coherence as much as possible.

---

### Cell 5 — Write Chunks to a Delta Table

**What it does:** Saves all chunks into a **Delta Table** in Unity Catalog.

```
rag_portfolio.doc_search.airflow_docs_chunks
├── id         (integer, primary key)
├── text       (the chunk text — this is what gets embedded)
├── source     (e.g. "airflow-docs/best-practices.html")
├── doc_type   (e.g. "airflow")
└── char_count (length of the chunk in characters)
```

**Why Delta?**
- Delta Lake supports ACID transactions — no partial writes
- It supports **Change Data Feed**, which Vector Search needs to sync automatically
- It is the native storage format across all Databricks services

---

### Cell 6 — Inspect the Delta Table

**What it does:** Runs a quick `SELECT` to verify the data was written correctly.

---

### Cell 7 — Enable Change Data Feed (CDF)

**What it does:** Turns on a Delta Lake feature that tracks every insert, update, and delete on the table.

```sql
ALTER TABLE rag_portfolio.doc_search.airflow_docs_chunks
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

**Why it matters:** Databricks Vector Search uses CDF to incrementally sync the index when new rows are added. Without CDF enabled, the Delta Sync Index cannot be created.

---

### Cell 8 — Create a Vector Search Endpoint

**What it does:** Provisions a **Vector Search compute endpoint** — a dedicated server that hosts and serves your vector index.

**What is an endpoint?**  
Think of it as a search engine server. It runs while active and handles similarity search queries against your index. This is the resource that incurs **hourly charges** — which is why Cell 19 (teardown) is critical.

**Wait time:** 5–10 minutes for provisioning.

---

### Cell 9 — Wait for the Endpoint to Come Online

**What it does:** Polls the Databricks API in a loop, checking `endpoint_status` every 30 seconds until it returns `"ONLINE"`.

This is a standard pattern when working with asynchronous cloud provisioning.

---

### Cell 10 — Create the Delta Sync Vector Index

**What it does:** Creates a **Delta Sync Index** on top of the `airflow_docs_chunks` Delta Table.

**What is a vector index?**  
Each chunk's `text` is sent through an **embedding model** that converts it into a list of numbers (a vector). These vectors are stored in the index. When you search, your query is also converted to a vector, and the index returns the chunks whose vectors are mathematically closest — i.e., semantically similar.

`databricks-gte-large-en` is a high-quality sentence embedding model hosted by Databricks Foundation Model APIs.

**Wait time:** 5–15 minutes for the initial embedding pass.

---

### Cell 11 — Wait for the Index to Be Ready

**What it does:** Polls the index status until it reports `"ONLINE"` with a non-zero row count.

---

### Cell 12 — Verify the Index

**What it does:** Checks the `num_rows_indexed` field. You should see `62` (or however many chunks were written in Cell 5).

---

### Cell 13 — Build the LangChain Retriever

**What it does:** Wraps the Vector Search index in a LangChain retriever object.

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 4}   # return top-4 most similar chunks
)
```

**What is a retriever?**  
In LangChain, a retriever is an object with a `.invoke(query)` method that returns relevant `Document` objects. `k=4` means: for each question, fetch the 4 most semantically relevant chunks.

---

### Cell 14 — Build the RAG Chain

**What it does:** Wires together the retriever and the LLM into a complete RAG chain using LangChain's pipe (`|`) syntax.

**How the chain works step-by-step:**
1. The user's question is passed to the `retriever` → returns 4 relevant chunks
2. The chunks are formatted as `context` and combined with the `question` into a prompt
3. The prompt is sent to `llama-4-maverick` via Foundation Model API
4. The LLM's text response is parsed and returned as a string

The system prompt instructs the LLM to answer only based on the provided context — preventing hallucination.

---

### Cell 15 — Run a Single Query

**What it does:** Invokes the chain with a test question and prints the answer. This is the moment the full pipeline runs end-to-end for the first time.

---

### Cell 16 — Batch Evaluation

**What it does:** Runs the chain against 5 test questions and collects results. This simulates a lightweight evaluation loop — a standard practice in production RAG systems.

---

### Cell 17 — Inspect Retrieved Context

**What it does:** Prints retrieved chunks and the final answer side-by-side for a selected question. Key debugging tool:

- **Good answer + good chunks** → pipeline is working correctly
- **Poor answer + good chunks** → LLM prompt or reasoning issue
- **Poor answer + poor chunks** → retrieval issue (chunking, embedding, or index quality)

---

### Cell 18 — Save Evaluation Results to Delta

**What it does:** Writes the batch evaluation results to `rag_portfolio.doc_search.eval_results` — a persistent, queryable audit trail of system performance.

---

### Cell 19 — ⚠ Teardown (ALWAYS RUN THIS)

**What it does:** Deletes the Vector Search index and the Vector Search endpoint.

**Why this is critical:**  
The Vector Search endpoint is billed **per hour** while active. Always run Cell 19 before closing your session.

---

## Common Issues and Fixes

| Issue | Likely Cause | Fix |
|---|---|---|
| `PERMISSION_DENIED` on CREATE CATALOG | Your user lacks catalog creation rights | Ask your workspace admin to grant `CREATE CATALOG` privilege |
| Endpoint stays in `PROVISIONING` for > 15 min | Workspace capacity limits | Try a different region or contact Databricks support |
| `Index not ONLINE` after 20 min | CDF not enabled before index creation | Re-run Cell 7 (enable CDF), then re-run Cell 10 |
| Kernel restarts after Cell 2 | Normal pip install behaviour | Expected — resume from Cell 3 |
| Empty answer from LLM | Query not in scraped docs | Expand the `pages` list in Cell 3 with more Airflow doc pages |
| `ValueError: num_rows_indexed = 0` | Index created before CDF was enabled | Delete index, re-enable CDF, recreate index |

---

## Concepts Glossary

| Term | Plain-English Definition |
|---|---|
| **Embedding** | A list of numbers that represents the meaning of a piece of text; similar texts have similar numbers |
| **Vector Search** | A search system that finds items by meaning (semantic similarity) rather than exact keyword matching |
| **Delta Table** | A reliable, versioned table stored in cloud storage; the native storage format in Databricks |
| **Unity Catalog** | Databricks' centralised data governance layer — manages who can access what data |
| **Change Data Feed (CDF)** | A Delta Lake feature that records every row change, enabling incremental processing |
| **LangChain** | A Python framework for building LLM-powered applications by chaining components together |
| **Retriever** | A LangChain component that fetches relevant documents given a query |
| **Foundation Model API** | Databricks-hosted LLMs and embedding models accessible via a simple API, no GPU management needed |
| **RAG** | Retrieval-Augmented Generation — using retrieved documents as context for LLM responses |
| **Chunk** | A fixed-size fragment of a document created by a text splitter |
| **k (in retrieval)** | The number of top matching chunks to return from a vector search query |

---

## Next Steps

Once you have run the full notebook successfully, consider these enhancements:

- **Add more data sources** — PDFs, Confluence pages, SharePoint, or your own internal docs
- **Tune chunk size** — experiment with 500, 750, 1500 character chunks and measure retrieval quality
- **Add re-ranking** — use a cross-encoder model to re-score retrieved chunks before passing to the LLM
- **Build a UI** — wrap the chain in a Gradio or Streamlit app for a chat interface
- **Add MLflow tracing** — Databricks supports end-to-end RAG tracing via MLflow for production observability
- **Automate ingestion** — replace the manual scraper with a Databricks Workflow triggered on a schedule

---

*Built by Ravi Amaraweera — Senior Data Architect / Analytics Engineer*  
*GitHub: [ramaraweera/databricks_rag_with_vectorSearch](https://github.com/ramaraweera/databricks_rag_with_vectorSearch)*
