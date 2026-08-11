# AI Document Intelligence Pipeline

This project is a production-grade RAG (Retrieval Augmented Generation) pipeline that transforms static PDF documents into a queryable knowledge base leveraging AI. Built to mirror the architecture of enterprise document processing platforms used in regulated industries.


## What It Does

This process allows you to upload any PDF into the pipeline to get accurate answers pulled directly from the document in approximately 1-3 seconds, at less than one cent per query.


## Architecture
PDF → S3 → PostgreSQL → Chunking → Embeddings → Pinecone → Anthropic Claude API → Answer

---

## Pipeline Stages

### Stage 1: Document Ingestion
- Generates MD5 hash fingerprint per document for automatic deduplication
- Uploads raw PDF to AWS S3 Buckets
- Writes metadata record to PostgreSQL tracking `doc_id`, `source_path`, `status`, `ingested_at`
- Duplicate documents are caught and skipped automatically via `ON CONFLICT DO NOTHING`

### Stage 2: Text Extraction
- Pulls PDF bytes directly from S3 into memory
- Extracts full text using PyMuPDF (fitz)
- Handles text-based PDFs natively

### Stage 3: Chunking
- Splits extracted text into 500-character overlapping chunks
- 100-character overlap prevents critical information loss at chunk boundaries
- Overlap ensures no sentence is orphaned between chunks

### Stage 4: Embedding
- Converts each chunk into a 384-dimension dense vector using `all-MiniLM-L6-v2`
- Model runs completely locally with zero API cost, zero external calls
- Same model used for both document chunks and query embedding at retrieval time

### Stage 5: Vector Storage
- Uploads all vectors to Pinecone with metadata: `doc_id`, `chunk_index`, `text`
- Index configured: dense vectors, cosine similarity, 384 dimensions
- Enables millisecond similarity search across any number of documents

### Stage 6: RAG Query Layer
- Embeds incoming question using same local model
- Queries Pinecone for top 5 semantically similar chunks
- Sends only relevant chunks as context to Claude Sonnet
- Claude returns grounded answer using only retrieved document content

## Why RAG Over Dumping the Whole File

Every query sends only the most relevant chunks, not the entire document. Because of this, RAG becomes increasingly cheaper relative to whole-document dumping in direct proportion to the size of each PDF. The larger the document, the wider the cost gap. Because RAG always retrieves only the most relevant chunks regardless of how large the source document is, while whole-document costs grow linearly with page count.

Beyond cost, there is a hard physical limit. A ~120,000 word page document contains approximately 250,000 tokens exceeding Claude's 200,000 token limit window entirely. Whole-document dumping becomes not just expensive but impossible at that scale. RAG has no such ceiling.

Approximations

1,000 words of plain English  = ~1,300 tokens
1,000 words of legal text     = ~1,600 tokens
200,000 token limit           = ~120,000-150,000 words

---

## Why This Architecture Matters for Regulated Industries

Insurance, healthcare, and financial services cannot route sensitive documents through third-party platforms like Microsoft Copilot or ChatGPT due to:

- **HIPAA** — Protected Health Information cannot touch external servers
- **Data sovereignty** — regulators require knowing exactly where customer data lives
- **Audit requirements** — every AI-generated answer must be traceable to a source document
- **Liability** — third-party data breaches expose the company equally

This Proof-of-Concept project keeps all data within your own infrastructure. Every answer is traceable to a specific chunk from a specific document. Compliance teams can verify exactly what the AI saw and why it answered the way it did.

---

## Cost of NOT Building This

| Manual Process | AI-Powered DocFlow |
|---|---|
| Analyst reads 200-page report manually | Query answered in 3 seconds |
| 30-45 minutes per question | < 1 cent per question |
| Knowledge locked in one person's head | Queryable by entire organization |
| New document = re-read everything | New document = rerun pipeline |
| No audit trail | Full chunk-level traceability |

---

## Tech Stack

| Layer | Tool |
|---|---|
| Cloud file storage | AWS S3 |
| Metadata database | PostgreSQL |
| PDF text extraction | PyMuPDF (fitz) |
| Embedding model | all-MiniLM-L6-v2 (local, free) |
| Vector database | Pinecone |
| LLM | Claude Sonnet (Anthropic) |
| Environment management | python-dotenv |
| Language | Python |

---

## Setup

**1. Clone the repo and install dependencies:**
```bash
pip install boto3 psycopg2 pymupdf sentence-transformers pinecone anthropic python-dotenv
```

**2. Configure your environment file (`rag_s3.env`):**
