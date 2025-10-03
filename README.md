DataMosaic🚀
AI-powered hybrid pipeline that converts unstructured documents into clean, structured data (CSV/DB) — Regex first, LLM fallback, embeddings-backed validation.


DataMosaic is a full-stack, production-minded project that extracts columns/fields from messy text/PDF/DOC files. It tries cheap deterministic methods first (regex), falls back to LLMs (Groq / Mistral) only when needed, and uses sentence-transformers + OpenAI nomadic-text embeddings to improve accuracy, validation and ranking.

Table of contents
[Why DataMosaic?](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#why-structifyai)
[Highlights / Features](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#highlights--features)
[Architecture (overview)](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#architecture-overview)
[Quick demo (example input → output)](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#quick-demo-example-input--output)
[Getting started (local)](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#getting-started-local)
[Backend API (endpoints & examples)](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#backend-api-endpoints--examples)
[Schema & LLM prompt examples](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#schema--llm-prompt-examples)
[Regex-first strategy & common patterns](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#regex-first-strategy--common-patterns)
[Embedding + sentence-transformer integration](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#embedding--sentence-transformer-integration)
[Token handling & chunking strategy](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#token-handling--chunking-strategy)
[Testing & sample data](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#testing--sample-data)
[Deployment suggestions](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#deployment-suggestions)
[Security, privacy & cost considerations](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#security-privacy--cost-considerations)
[Roadmap & future improvements](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#roadmap--future-improvements)
[Contributing](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#contributing)
[License & credits](https://chatgpt.com/c/68dfd976-e488-8324-a1df-ad9307be85f8#license--credits)

Why DataMosaic?
Many teams face a recurring problem: mission-critical information sits locked in messy documents (reports, bank statements, police reports, emails). DataMosaic solves it by combining three approaches into a single pipeline so solutions are accurate, fast, and cost-effective:
Regex-first for predictable / cheap extractions.
LLM fallback (Groq preferred; Mistral fallback) for ambiguous/complex cases.
Embeddings + Sentence Transformers (all-mini-lm-v6 + nomadic-text) to select the most relevant text chunks and validate LLM outputs.
This hybrid approach reduces LLM usage (cutting cost) while keeping flexibility and accuracy for real-world documents.

Highlights / Features
User-defined schema: upload files, specify column names (e.g., Name, DOB, Amount, CaseID) and get a filled table.
Regex-first extraction, then intelligent fallback to LLM if any column is missing/confidently wrong.
LLM integration designed for Groq inference API (4096 token input/output). Auto-fallback to Mistral if Groq unavailable.
Embedding-based semantic selection: use all-mini-lm-v6 locally to find relevant segments and nomadic-text for higher-quality embeddings.
Output formats: CSV download, SQLite/Postgres storage, JSON API.
Modern full-stack: FastAPI backend + React + Tailwind frontend skeleton included.
Logging, confidence scoring, and human-in-the-loop corrections support (trainer UI planned).

Architecture (overview)
[User Uploads File / Specifies Schema]
                ↓
        File Processing
        (PDF/DOC/TXT/OCR)
                ↓
       Text Segmentation / Chunking
                ↓
Regex Extraction (fast)  ──→ success? ──→ Map to columns & return
         │
         └─ no → Select relevant chunks via embeddings (all-mini-lm-v6)
                       ↓
             LLM Fallback (Groq / Mistral)
                       ↓
        Merge LLM + Regex results → Post-validate with embeddings
                       ↓
        Store result (CSV / DB) + UI display


Quick demo (example input → output)
Input (unstructured text snippet):
Invoice # INV-2025-0045
Date: 21/09/2025
Billed To: Rahul Sharma
Phone: +91 9876543210
Total: ₹12,350.50

User Schema (columns):
["InvoiceID", "Date", "CustomerName", "Phone", "TotalAmount"]

Processing
Regex finds InvoiceID, Date, Phone, Total.
CustomerName absent from regex? If regex fails, LLM is asked to extract CustomerName from the chunk(s).
Embedding verifies the location & confidence, then stores results.
Output (CSV row):
INV-2025-0045,2025-09-21,Rahul Sharma,+91 9876543210,12350.50


Getting started (local)
Prerequisites
Python 3.10+ (venv recommended)
Node.js 18+ / npm or yarn
pip packages (listed below)
API keys for Groq / Mistral and OpenAI (for nomadic-text), added to .env file
Clone
git clone [https://github.com/your-username/DataMosaic.git](https://github.com/your-username/DataMosaic.git)
cd DataMosaic

Backend (FastAPI)
cd backend
python -m venv .venv
source .venv/bin/activate        # on Windows: .venv\Scripts\activate
pip install -r requirements.txt
# Start server (dev)
uvicorn main:app --reload --port 8000

Environment variables (.env)
GROQ_API_KEY=your_groq_key
MISTRAL_API_KEY=your_mistral_key
OPENAI_API_KEY=your_openai_key
EMBEDDING_PROVIDER=local  # or openai
DATABASE_URL=sqlite:///./data.db
MAX_TOKENS=4096

Frontend (React + Tailwind)
cd ../frontend
npm install
npm run dev
# or yarn dev

Open http://localhost:3000 to access the UI.

Backend API (endpoints & examples)
1. POST /api/upload
Upload file(s) and define schema.
Request (multipart/form-data):
file: file (pdf, docx, txt)
schema: JSON array string of column names
cURL
curl -X POST "http://localhost:8000/api/upload" \
  -F "file=@/path/to/doc.pdf" \
  -F 'schema=["InvoiceID","Date","Name","Phone","Amount"]'

Response
{
  "upload_id": "uuid-1234",
  "status": "uploaded",
  "schema": ["InvoiceID","Date","Name","Phone","Amount"]
}

2. POST /api/extract
Trigger extraction (sync or queued async).
Request (JSON):
{
  "upload_id": "uuid-1234",
  "options": {
    "regex_first": true,
    "llm_model": "groq-4096",
    "embedding_provider": "all-mini-lm-v6"
  }
}

Response (sample)
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

3. GET /api/result/{upload_id}
Download CSV / view JSON result.
cURL (download CSV)
curl -o result.csv "http://localhost:8000/api/result/uuid-1234?format=csv"


Schema & LLM prompt examples
Schema format
You can accept simple arrays or typed schemas with validation:
{
  "columns": [
    {"name": "InvoiceID", "type": "string"},
    {"name": "Date", "type": "date", "formats": ["%d/%m/%Y","%Y-%m-%d"]},
    {"name": "Phone", "type": "phone"},
    {"name": "Amount", "type": "currency"}
  ]
}

LLM prompt template (example)
You are an information extraction assistant. Given the following text and the target schema, extract values for each field.
Text:
---
{TEXT_CHUNK}
---
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

Example prompt usage notes
Send only the relevant chunk(s); avoid sending the entire document when unnecessary.
Include a few examples (1–3) in the prompt if you want stronger guidance (few-shot), but watch token limits.
Ask the LLM to return a confidence score per field (0–1) — many LLMs can approximate this.

Regex-first strategy & common patterns
Start with deterministic patterns for well-known fields. If a regex returns a valid result, mark as CONFIDENT and avoid LLM.
Common patterns
regex_patterns = {
  "email": r"[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+",
  "phone_in_india": r"(?:\+91[\-\s]?)?[6-9]\d{9}",
  "date_ddmmyyyy": r"\b(0?[1-9]|[12][0-9]|3[01])[\/\-.](0?[1-9]|1[012])[\/\-.](\d{4})\b",
  "invoice_id": r"\bINV[-_ ]?\d{3,7}\b",
  "amount": r"[₹$€]?\s?[\d{1,3}(?:,\d{3})*(?:\.\d+)?]+"
}

Algorithm
For each column, run matched relevant regex patterns from a library.
If match and type-valid, mark column DONE and store a confidence (regex confidence = 0.9+ by default).
For missing/low-confidence columns, proceed to embeddings selection + LLM extraction.

Embedding + sentence-transformer integration
Why embeddings?
To select the most relevant chunk(s) for each target column instead of sending full docs to the LLM.
To validate the LLM output by measuring semantic similarity between the extracted value and the source chunk.
Recommended models & flow
Use sentence-transformers/all-mini-lm-v6 locally for fast vectorization of document chunks and the query (column description).
Optionally compute nomadic-text embeddings via OpenAI for production-grade matching (higher upstream cost).
Use a small vector index (FAISS / in-memory) to retrieve top-K chunks per column (K=1..3).
Send those chunks (concatenated, with separators) to LLM for column extraction.
Example retrieval pseudo-flow
# compute embedding for column description
q_emb = embed("Customer name for invoice")
# compute similarities vs chunk embeddings
top_chunks = faiss.search(q_emb, k=3)
# send top_chunks to LLM


Token handling & chunking strategy
Goal: respect 4096 token limit (input + output) for Groq (or chosen LLM).
Chunking approach
Split text into logical paragraphs or ~512–1000 token chunks.
Keep chunk metadata: page_number, char_offsets.
For each column, retrieve top-k chunks by embedding relevance and pass only those to the LLM.
If the chosen LLM returns truncated output or TOO_LONG, reduce included chunks.
Output sizing
Ask LLM for concise JSON (no extraneous text).
Set a max token output param; if too small, the model may truncate — test empirically.

Testing & sample data
Provide small dataset of sample docs: receipts, invoices, police report text, bank statement snippets, and .txt versions.
Unit tests for:
Regex extraction correctness (multiple formats).
Embedding selection (relevance ranking).
LLM prompt parsing & JSON validation.
Integration tests:
End-to-end flow: upload → extract → CSV export.
Add tests to assert that regex-first avoids LLM calls when possible.

Deployment suggestions
Dockerize backend & frontend. Minimal docker-compose for local run.
For quick hosting:
Backend: Render / Railway / Heroku (or cloud VM). Use environment variables for API keys.
Frontend: Vercel / Netlify.
Production considerations:
Use managed vector DB (Pinecone / Weaviate) if scale needed.
Use a job queue (Redis + RQ / Celery) for large/async extractions.
Metrics & cost tracking (Prometheus + Grafana or cloud monitoring).

Security, privacy & cost considerations
Security / Privacy
Securely store API keys; do not commit .env. Use secret managers in production.
If documents contain PII, consider:
Masking outputs.
Providing on-prem or VPC-limited deployment.
Audit logs for human review.
Cost optimization
Regex-first avoids unnecessary LLM calls.
Cache embeddings & LLM responses for repeated docs.
Batch similar jobs to reuse context when possible.
Track LLM token usage and set alerts.

Roadmap & future improvements
Phase 1 (MVP)
FastAPI backend with:
file upload
regex engine
LLM fallback integration for missing fields
CSV output
React frontend skeleton
Phase 2
Embedding retrieval (all-mini-lm-v6 + nomadic-text)
Confidence scoring & UI for human verification
Batch processing & queueing system
Phase 3
OCR pipeline for scanned PDFs (Tesseract / cloud OCR)
Interactive trainer (users correct outputs; system learns patterns)
Multi-language support and domain-specific templates
Enterprise features: auth, workspaces, audit logs

Contributing
Fork the repo → create feature branch → raise PR.
Please include tests for new features.
Add clear descriptions for configuration changes.

License & credits
License: MIT (feel free to change based on your needs).
Credits: Many open-source projects and papers inspired the design:
kor, llm-ie, ExtractThinker, ContextGem, LangExtract and others.
Hugging Face sentence-transformers and OpenAI embeddings.

Final notes — TL;DR
DataMosaic is positioned as a practical, production-friendly system for turning messy documents into usable tables. Its main advantages are cost-efficiency (regex-first), robustness (LLM fallback), and accuracy (embeddings-guided chunk selection and post-validation). It’s a portfolio-grade project: full-stack, technically interesting, and valuable to many industries.
