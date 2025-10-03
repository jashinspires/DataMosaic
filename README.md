# DataMosaic

AI-powered hybrid pipeline that converts unstructured documents into clean, structured data (CSV/DB) — regex-first extraction, LLM fallback, and embeddings-backed validation.

DataMosaic is a full-stack, production-minded project that extracts columns/fields from messy text/PDF/DOC files. It prioritizes deterministic methods (regex), falls back to LLMs (Groq / Mistral) only when needed, and uses sentence-transformers plus OpenAI nomadic-text embeddings to improve accuracy, validation, and ranking.

***

## Table of Contents
- [Why DataMosaic?](#why-datamosaic)
- [Highlights / Features](#highlights--features)
- [Architecture (Overview)](#architecture-overview)
- [Quick Demo (Example Input → Output)](#quick-demo-example-input--output)
- [Getting Started (Local)](#getting-started-local)
  - [Backend (FastAPI)](#backend-fastapi)
  - [Start Server (Dev)](#start-server-dev)
  - [Environment Variables](#environment-variables)
  - [Frontend (React + Tailwind)](#frontend-react--tailwind)
- [Backend API (Endpoints & Examples)](#backend-api-endpoints--examples)
- [Schema & LLM Prompt Examples](#schema--llm-prompt-examples)
- [Regex-first Strategy & Common Patterns](#regex-first-strategy--common-patterns)
- [Embedding + Sentence-Transformer Integration](#embedding--sentence-transformer-integration)
- [Token Handling & Chunking Strategy](#token-handling--chunking-strategy)
- [Testing & Sample Data](#testing--sample-data)
- [Deployment Suggestions](#deployment-suggestions)
- [Security, Privacy & Cost Considerations](#security-privacy--cost-considerations)
- [Roadmap & Future Improvements](#roadmap--future-improvements)
- [Contributing](#contributing)
- [License & Credits](#license--credits)

***

## Why DataMosaic?
Many teams face a recurring problem: mission-critical information sits locked in messy documents (reports, bank statements, police reports, emails). DataMosaic solves this by combining three approaches into a single pipeline for accuracy, speed, and cost-effectiveness:

- Regex-first for predictable, cheap extractions
- LLM fallback (Groq preferred; Mistral fallback) for ambiguous/complex cases
- Embeddings + Sentence Transformers (all-mini-lm-v6 + nomadic-text) to select relevant text chunks and validate LLM outputs

This hybrid approach reduces LLM usage (cutting cost) while retaining flexibility and accuracy for real-world documents.

***

## Highlights / Features
- User-defined schema: upload files, specify column names (e.g., Name, DOB, Amount, CaseID) and get a filled table
- Regex-first extraction, then intelligent fallback to LLM if any column is missing or low-confidence
- LLM integration designed for Groq inference API (4096 token input/output), auto-fallback to Mistral if Groq unavailable
- Embedding-based semantic selection: use all-mini-lm-v6 locally and nomadic-text for higher-quality embeddings
- Output formats: CSV download, SQLite/Postgres storage, JSON API
- Modern full-stack: FastAPI backend + React + Tailwind frontend skeleton included
- Logging, confidence scoring, and human-in-the-loop corrections support (trainer UI planned)

***

## Architecture (Overview)

**Flow:**

1. User Uploads File / Specifies Schema  
2. File Processing (PDF/DOC/TXT/OCR)  
3. Text Segmentation / Chunking  
4. Regex Extraction (fast)  
   - If success → Map to columns & return  
   - If not → Select relevant chunks via embeddings (all-mini-lm-v6)  
5. LLM Fallback (Groq / Mistral)  
6. Merge LLM + Regex results → Post-validate with embeddings  
7. Store result (CSV / DB) + UI display  

**Pipeline Architecture (ASCII Version)**

User Uploads File / Schema
|
v
File Processing (PDF/DOC/TXT)
|
v
Text Segmentation / Chunking
|
v
Regex Extraction
/
Yes No
| |
v v
Merge + LLM Embeddings: Select
| Relevant Chunks
| |
+-------+-------+
|
v
LLM Fallback
|
v
Merge + Validate
|
v
Store CSV/DB + UI

***

## Quick Demo (Example Input → Output)

Input (unstructured text snippet):
```
Invoice # INV-2025-0045
Date: 21/09/2025
Billed To: Rahul Sharma
Phone: +91 9876543210
Total: ₹12,350.50
```

User Schema (columns):
```json
["InvoiceID", "Date", "CustomerName", "Phone", "TotalAmount"]
```

Processing:
- Regex finds InvoiceID, Date, Phone, Total
- CustomerName absent from regex → LLM extracts CustomerName from top-ranked chunk(s)
- Embedding verifies location & confidence, then stores results

Output (CSV row):
```
INV-2025-0045,2025-09-21,Rahul Sharma,+91 9876543210,12350.50
```

***

## Getting Started (Local)

### Prerequisites
- Python 3.10+ (venv recommended)
- Node.js 18+ / npm or yarn
- pip packages (listed below)
- API keys for Groq / Mistral and OpenAI (for nomadic-text), added to .env file

### Clone
```bash
git clone https://github.com/your-username/DataMosaic.git
cd DataMosaic
```

### Backend (FastAPI)
```bash
cd backend
python -m venv .venv
source .venv/bin/activate  # on Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### Start Server (Dev)
```bash
uvicorn main:app --reload --port 8000
```

### Environment Variables
Create a .env file in the backend directory:
```env
GROQ_API_KEY=your_groq_key
MISTRAL_API_KEY=your_mistral_key
OPENAI_API_KEY=your_openai_key
EMBEDDING_PROVIDER=local  # or openai
DATABASE_URL=sqlite:///./data.db
MAX_TOKENS=4096
```

### Frontend (React + Tailwind)
```bash
cd ../frontend
npm install
npm run dev
# or
yarn dev
```

Open http://localhost:3000 to access the UI.

***

## Backend API (Endpoints & Examples)

1. POST /api/upload — Upload file(s) and define schema
   - Request (multipart/form-data):
     - file: file (pdf, docx, txt)
     - schema: JSON array string of column names
   - cURL
     ```bash
     curl -X POST "http://localhost:8000/api/upload" \
       -F "file=@/path/to/doc.pdf" \
       -F 'schema=["InvoiceID","Date","Name","Phone","Amount"]'
     ```
   - Response
     ```json
     {
       "upload_id": "uuid-1234",
       "status": "uploaded",
       "schema": ["InvoiceID","Date","Name","Phone","Amount"]
     }
     ```

2. POST /api/extract — Trigger extraction (sync or queued async)
   - Request (JSON)
     ```json
     {
       "upload_id": "uuid-1234",
       "options": {
         "regex_first": true,
         "llm_model": "groq-4096",
         "embedding_provider": "all-mini-lm-v6"
       }
     }
     ```
   - Response (sample)
     ```json
     {
       "upload_id": "uuid-1234",
       "structured": {
         "InvoiceID": "INV-2025-45",
         "Date": "2025-09-21",
         "Name": "Rahul Sharma",
         "Phone": "+91 9876543210",
         "Amount": 12350.5
       },
       "confidence": {
         "InvoiceID": 0.98,
         "Date": 0.99,
         "Name": 0.85,
         "Phone": 0.99,
         "Amount": 0.97
       },
       "used_regex": ["InvoiceID","Date","Phone","Amount"],
       "used_llm": ["Name"]
     }
     ```

3. GET /api/result/{upload_id} — Download CSV / view JSON result
   - cURL (download CSV)
     ```bash
     curl -o result.csv "http://localhost:8000/api/result/uuid-1234?format=csv"
     ```

***

## Schema & LLM Prompt Examples

### Schema Format
You can accept simple arrays or typed schemas with validation:
```json
{
  "columns": [
    {"name": "InvoiceID", "type": "string"},
    {"name": "Date", "type": "date", "formats": ["%d/%m/%Y","%Y-%m-%d"]},
    {"name": "Phone", "type": "phone"},
    {"name": "Amount", "type": "currency"}
  ]
}
```

### LLM Prompt Template (example)
```
You are an information extraction assistant. Given the following text and the target schema, extract values for each field.

Text:
{TEXT_CHUNK}

Schema: {SCHEMA_JSON}

Rules:
- Output JSON only.
- If a field cannot be confidently identified, return null for that field.
- Normalize dates to ISO 8601 (YYYY-MM-DD).
- Normalize currencies to numeric amount (no currency symbol).

Return:
{
  "InvoiceID": "...",
  "Date": "...",
  "Name": "...",
  "Phone": "...",
  "Amount": ...
}
```

Example prompt usage notes:
- Send only the relevant chunk(s); avoid sending the entire document when unnecessary
- Include a few examples (1–3) in the prompt if you want stronger guidance (few-shot), but watch token limits
- Ask the LLM to return a confidence score per field (0–1)

***

## Regex-first Strategy & Common Patterns
Start with deterministic patterns for well-known fields. If a regex returns a valid result, mark as CONFIDENT and avoid LLM.

Common patterns:
```python
regex_patterns = {
  "email": r"[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+",
  "phone_in_india": r"(?:\+91[-\s]?)?[6-9]\d{9}",
  "date_ddmmyyyy": r"\b(0?[1-9]|[12][0-9]|3[01])[/\-.](0?[1-9]|1[012])[/\-.]\d{4}\b",
  "invoice_id": r"\bINV[-_ ]?\d{3,7}\b",
  "amount": r"[₹$€]?\s?[\d{1,3}(?:,\d{3})*(?:\.\d+)?]+"
}
```

Algorithm:
1. For each column, run matched relevant regex patterns from a library
2. If match and type-valid, mark column DONE and store a confidence (regex confidence = 0.9+ by default)
3. For missing/low-confidence columns, proceed to embeddings selection + LLM extraction

***

## Embedding + Sentence-Transformer Integration

Why embeddings?
- Select the most relevant chunk(s) for each target column instead of sending full docs to the LLM
- Validate the LLM output by measuring semantic similarity between the extracted value and the source chunk

Recommended models & flow:
- Use sentence-transformers/all-mini-lm-v6 locally for fast vectorization
- Optionally compute nomadic-text embeddings via OpenAI for production-grade matching
- Use a small vector index (FAISS / in-memory) to retrieve top-K chunks per column (K=1..3)
- Send those chunks (concatenated, with separators) to LLM for column extraction

Example retrieval pseudo-flow:
```python
# compute embedding for column description
q_emb = embed("Customer name for invoice")

# compute similarities vs chunk embeddings
top_chunks = faiss.search(q_emb, k=3)

# send top_chunks to LLM
```

***

## Token Handling & Chunking Strategy
Goal: respect 4096 token limit (input + output) for Groq (or chosen LLM).

Chunking approach:
- Split text into logical paragraphs or ~512–1000 token chunks
- Keep chunk metadata: page_number, char_offsets
- For each column, retrieve top-k chunks by embedding relevance and pass only those to the LLM
- If the chosen LLM returns truncated output or TOO_LONG, reduce included chunks

Output sizing:
- Ask LLM for concise JSON (no extraneous text)
- Set a max token output param; test empirically to avoid truncation

***

## Testing & Sample Data
Provide a small dataset of sample docs: receipts, invoices, police report text, bank statement snippets, and .txt versions.

Unit tests for:
- Regex extraction correctness (multiple formats)
- Embedding selection (relevance ranking)
- LLM prompt parsing & JSON validation

Integration tests:
- End-to-end flow: upload → extract → CSV export
- Assert that regex-first avoids LLM calls when possible

***

## Deployment Suggestions
- Dockerize backend & frontend (minimal docker-compose for local run)
- Quick hosting: Backend (Render / Railway / Heroku or a cloud VM); Frontend (Vercel / Netlify)
- Production considerations:
  - Managed vector DB (Pinecone / Weaviate) if scale needed
  - Job queue (Redis + RQ / Celery) for large/async extractions
  - Metrics & cost tracking (Prometheus + Grafana or cloud monitoring)

***

## Security, Privacy & Cost Considerations
Security / Privacy:
- Securely store API keys; do not commit .env
- Use secret managers in production
- If documents contain PII, consider masking outputs, on-prem/VPC deployment, and audit logs for human review

Cost optimization:
- Regex-first avoids unnecessary LLM calls
- Cache embeddings & LLM responses for repeated docs
- Batch similar jobs to reuse context when possible
- Track LLM token usage and set alerts

***

## Roadmap & Future Improvements
- Phase 1 (MVP)
  - FastAPI backend with: file upload, regex engine, LLM fallback integration for missing fields, CSV output
  - React frontend skeleton
- Phase 2
  - Embedding retrieval (all-mini-lm-v6 + nomadic-text), confidence scoring & UI for human verification, batch processing & queueing
- Phase 3
  - OCR pipeline for scanned PDFs (Tesseract / cloud OCR)
  - Interactive trainer (users correct outputs; system learns patterns)
  - Multi-language support and domain-specific templates
  - Enterprise features: auth, workspaces, audit logs

***

## Contributing
Contributions are welcome!

- Fork the repo → create a feature branch → raise a PR
- Please include tests for new features
- Add clear descriptions for configuration changes

***

## License & Credits
- License: MIT (feel free to change based on your needs)
- Credits: Many open-source projects and papers inspired the design: kor, llm-ie, ExtractThinker, ContextGem, LangExtract, and others. Hugging Face sentence-transformers and OpenAI embeddings.

***

## Final Notes — TL;DR
DataMosaic is a practical, production-friendly system for turning messy documents into usable tables. Its advantages are cost-efficiency (regex-first), robustness (LLM fallback), and accuracy (embeddings-guided chunk selection and post-validation). It’s a portfolio-grade project: full-stack, technically interesting, and valuable to many industries.
