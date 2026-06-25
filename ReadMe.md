# ⚖️ Legal Research Tool 
A RAG system for legal research, built to run in **Google Colab**. It loads legal documents into a vector database, then answers questions with inline citations, flags regulatory changes, and auto-drafts **legal briefs** — all grounded strictly in the documents you give it.

## 1. Purpose
Legal research means digging through case law, compliance policies, and regulatory updates to find what's relevant and trustworthy. A general chatbot will happily invent case names and citations, which is dangerous in a legal setting.

This project solves that by combining retrieval with generation:
- It only answers from the documents you ingest — if the answer isn't in the knowledge base, it says so instead of guessing.
- Every factual claim is cited inline as `[Source N]`, so each statement can be traced back to a source document.
- It is explicitly instructed never to fabricate** case names, statutes, or citations.
The result is a research assistant that helps you query legal material, spot regulatory changes that need attention, and draft structured briefs — while keeping every answer anchored to real sources.

## 2. How It Works
The system follows a standard RAG pipeline across three stages:

1. Ingestion — Each document is loaded, cleaned, and split into overlapping chunks (~500 words with 100-word overlap). Each chunk is converted into a vector embedding using `sentence-transformers` (`all-MiniLM-L6-v2`) and stored in **ChromaDB** with metadata (jurisdiction, year, effective date, regulator, etc.). Documents are organized into three separate collections:
   - `case_law` — court cases and legal briefs
   - `compliance` — compliance policies
   - `regulatory` — regulatory updates

2. Retrieval — When you ask a question, it is embedded the same way and compared against the relevant collections. The closest-matching chunks (top-k, ranked by distance) are pulled out as context.

3. Generation — The retrieved chunks are formatted as numbered context and sent to the **Groq** LLM (`llama-3.1-8b-instant`) under a system prompt that enforces:
   - inline `[Source N]` citations,
   - a fixed answer structure (Summary → Legal Analysis → Key Precedents → Caveats),
   - a refusal to answer when the context doesn't support it.

Everything is exposed through a single helper, `legal_assistant(query, mode=...)`, with three modes:

| Mode | What it does |
|------|--------------|
| `query` | Answers a legal question with cited sources |
| `regulatory` | Scans for regulatory changes on a topic and assigns urgency (`IMMEDIATE` / `REVIEW IN 90 DAYS` / `MONITOR`) |
| `brief` | Drafts a structured legal brief from a set of case facts |

> An optional **LangChain** pipeline is also included as an alternative way to handle ingestion and the RAG chain, using `ChatGroq` and the Chroma vector store wrapper.

## 3. Documents Used
The notebook automatically generates 4 sample `.txt` files to populate the database so you can run it end-to-end without uploading anything:

| File | Type | Content |
|------|------|---------|
| `roe_v_wade.txt` | Case law | Supreme Court decision summary and precedent |
| `gdpr_compliance.txt` | Compliance | GDPR data-retention and breach-notification policy |
| `sec_ruling_2024.txt` | Regulatory | SEC cybersecurity disclosure rule |
| `employment_discrimination.txt` | Case law / brief | Title VII employment-discrimination brief |

In addition, you can bring in your **own reference documents**:

- **1 PDF** — parsed with PyPDF2 (e.g., `legaldoc.pdf`)
- **1 Word document** — parsed with python-docx (e.g., `legaldoc1.docx`)
Upload your file in Colab, then register it with the matching helper:

```python
add_case_law("legaldoc.pdf",  jurisdiction="US Federal", year="2024")     # PDF
add_case_law_docx("legaldoc1.docx", jurisdiction="US Federal", year="2024") # Word doc

# Other helpers:
add_compliance_policy("policy.pdf", effective_date="2024-06-01")
add_regulatory_update("ftc_rule.pdf", regulator="FTC", effective_date="2024-03-15")
```

Supported formats overall: `.txt`, `.md`, `.pdf`, and `.docx`.

## 4. How to Run It
This notebook is designed to run on Google Colab with a Groq API key, using ChromaDB as the vector database.

1. **Open the notebook in Google Colab.**

2. **Install dependencies** (Cell 1):
   ```python
   !pip install -q groq chromadb sentence-transformers PyPDF2 python-docx numpy
   # Optional (LangChain pipeline):
   !pip install -q langchain langchain-community langchain-groq
   ```

3. **Add your Groq API key.** Get one free from [console.groq.com](https://console.groq.com). When you run the config cell, paste it at the prompt (it's read securely via `getpass`, so it isn't saved in the notebook).

4. **Run the cells in order.** The notebook will:
   - initialize the Groq client and the three ChromaDB collections,
   - create the 4 sample `.txt` documents,
   - embed and ingest them.

5. **(Optional) Upload your own documents** using the Colab file browser on the left, then register them with the helpers shown above. Colab uploads land in `/content/`, so reference them as e.g. `/content/legaldoc.pdf`.

6. **Ask questions:**
   ```python
   legal_assistant("What are the precedents for racial discrimination in employment?", mode="query")
   legal_assistant("cybersecurity disclosure requirements for public companies", mode="regulatory")
   legal_assistant("Employee was fired because of their race.", mode="brief")
   ```


This is a research and educational prototype. Outputs are generated by an LLM over a limited document set and **are not legal advice**. Always verify citations and conclusions against primary sources, and consult a qualified attorney for any actual legal matter.
