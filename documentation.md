# BidGenius – System Design Document

**Date:** 26 February 2026  

---

## 1. Executive Overview

BidGenius is an AI-powered bid management platform built for BluSapphire Cyber Systems. It automates two core workflows for government tender responses:

| Workflow | What It Does |
|----------|-------------|
| **Proposal Generation** | Ingests RFP documents → extracts requirements → generates a complete technical proposal as a Word document |
| **PQ Auto-Fill** | Takes a Pre-Qualification Excel questionnaire → queries a knowledge base → fills compliance and remarks columns using AI |

The system is deployed as a single Docker container on Railway with a persistent volume for data, and also runs as a Tauri desktop app for local use.

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    Frontend (Vite + React)           │
│  Dashboard │ Bid Info │ Triage │ Generate │ PQ │     │
└────────────────────────┬────────────────────────────┘
                         │ REST API (JSON)
┌────────────────────────▼────────────────────────────┐
│                  FastAPI Backend                     │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │Ingestion │  │Generation│  │ PQ       │          │
│  │ Worker   │  │ Worker   │  │ Worker   │          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘          │
│       │              │             │                │
│  ┌────▼──────────────▼─────────────▼────────────┐   │
│  │          Service Layer                       │   │
│  │  ingestion_service │ proposal_orchestrator   │   │
│  │  triage_service    │ pq_service              │   │
│  └──────────┬────────────────────┬──────────────┘   │
│             │                    │                  │
│  ┌──────────▼──────┐  ┌─────────▼──────────┐       │
│  │   SQLite (DB)   │  │  ChromaDB (Vector) │       │
│  │  Bids, Jobs,    │  │  Bid docs (RAG)    │       │
│  │  Proposals,     │  │  PQ KB (semantic)  │       │
│  │  Activity Logs  │  │                    │       │
│  └─────────────────┘  └────────────────────┘       │
└─────────────────────────────────────────────────────┘
```

### 2.1 Technology Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui |
| Backend | Python 3.11, FastAPI, SQLAlchemy (async), Pydantic |
| AI/LLM | AutoGen agents, OpenRouter API (xiaomi/mimo-v2-flash) |
| Vector DB | ChromaDB (persistent) with OpenAI-compatible embeddings (baai/bge-m3) |
| Database | SQLite (async via aiosqlite) |
| Document Parsing | unstructured, PyPDF2, openpyxl |
| Document Output | python-docx (Word generation) |
| Desktop | Tauri v2 (Rust shell) |
| Deployment | Docker → Railway (with persistent volume at `/data`) |

---

## 3. Data Model

```
┌──────────┐     ┌────────────┐     ┌────────────────┐
│   Bid    │──1:N│  Document  │──1:1│ IngestionJob   │
│          │     │            │     │                │
│ id       │     │ bid_id (FK)│     │ document_id(FK)│
│ title    │     │ filename   │     │ status         │
│ status   │     │ path       │     │ progress       │
│ bidNumber│     │ status     │     │ message        │
└────┬─────┘     └────────────┘     └────────────────┘
     │
     ├──1:N──┬──────────────┐     ┌─────────────────┐
     │       │ Requirement  │     │  TriageRun      │
     │       │ text         │     │  bid_id (FK)    │
     │       │ category     │     │  status         │
     │       │ priority     │     └─────────────────┘
     │       └──────────────┘
     │
     ├──1:1──┬──────────────┐     ┌─────────────────┐
     │       │  Proposal    │     │ GenerationJob   │
     │       │  content(JSON│     │ bid_id (FK)     │
     │       │  status      │     │ status/progress │
     │       └──────────────┘     └─────────────────┘
     │
     ├──1:N──┬──────────────┐
     │       │  RFPFacts    │
     │       │  bid_number  │
     │       │  buyer_org   │
     │       │  emd_amount  │
     │       └──────────────┘
     │
     └──1:N──┬──────────────┐
             │TriageSignal  │
             │ question_key │
             │ answer_text  │
             └──────────────┘

┌──────────┐     ┌────────────────┐
│  PQJob   │     │ PQKBDocument   │
│ input_path│    │ filename       │
│ status   │     │ qa_count       │
│ log_text │     │ status         │
└──────────┘     └────────────────┘

┌──────────────┐
│ ActivityLog  │  (cross-cutting: tracks all worker activity)
│ job_type     │
│ job_id       │
│ level        │
│ title        │
│ message      │
│ step_number  │
└──────────────┘
```

---

## 4. Proposal Generation Pipeline (Bid Flow)

This is the primary workflow. A user uploads an RFP document and the system generates a complete technical proposal.

### 4.1 End-to-End Flow

```
Upload PDF  →  Ingest  →  Triage  →  Generate  →  Export DOCX
```

### 4.2 Stage 1: Document Ingestion

**Entry:** `POST /api/v1/bids/{bid_id}/documents/upload`  
**Code:** `ingestion_service.py` → `doc_collector.py` → `parser.py` → ChromaDB

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐     ┌──────────┐
│ Upload PDF  │────▶│ Collect      │────▶│ Parse &     │────▶│ Embed in │
│ (save to    │     │ linked docs  │     │ chunk       │     │ ChromaDB │
│  disk)      │     │ from PDF     │     │ (semantic)  │     │          │
└─────────────┘     └──────────────┘     └─────────────┘     └──────────┘
```

#### Document Collection (`doc_collector.py`)
- Extracts hyperlinks from the uploaded PDF (PyPDF2)
- Downloads linked documents (.pdf, .docx, .xlsx) into the same directory
- Filters by allowed extensions, deduplicates by filename
- Non-fatal: if a download fails (e.g., government site geo-blocked), it logs a warning and continues

#### Table-Aware Semantic Chunking (`parser.py`)
This is a key innovation. Standard chunking loses table context. Our parser:

1. **Extracts elements** using `unstructured` library (text + tables)
2. **Detects table regions** using heuristics (short text, no verbs, contains numbers/specs)
3. **Applies accumulation rules:**
   - **Rule A:** Detect table regions via short text without verbs
   - **Rule B:** First title before table pattern = table name
   - **Rule C:** Each table row becomes: `"<Table Name> – <Label>: <Value>"`
   - **Rule D:** Never merge table rows with prose
4. **Produces chunks** with metadata: `bid_id`, `document_id`, `page`, `chunk_type`, `chunk_index`

Each chunk is typically 80–600 characters, semantically coherent, and suitable for retrieval.

#### Embedding
- Chunks are embedded into ChromaDB using `baai/bge-m3` embeddings via OpenRouter
- Collection name: `bid_docs_v2`
- Stored per-bid with metadata for filtering

### 4.3 Stage 2: Triage

**Entry:** `POST /api/v1/triage/{bid_id}/run` or manual via Triage UI  
**Code:** `triage_worker.py` → `triage_extractor.py` → `coverage_service.py`

Triage does three things:

#### a) Requirement Extraction
- Fetches all embedded chunks for the bid from ChromaDB
- Processes chunks in batches of 4
- Uses an AutoGen LLM agent (`triage_extractor`) to extract structured requirements from each batch
- Each requirement has: `text`, `category` (mandatory/desirable/eligibility), `priority`
- Results stored in `requirements` table

#### b) Coverage Analysis
- After requirement extraction, runs `coverage_service.py`
- For each requirement, searches ChromaDB for supporting chunks from company documents
- Classifies as **covered** (evidence found) or **gap** (no evidence)
- Produces a `CoverageSnapshot` with total/covered counts, gaps list, and strengths with evidence
- This tells the user which requirements BluSapphire can address vs. where gaps exist

#### c) RFP Facts Extraction
- Extracts structured metadata from the RFP using RAG + LLM
- Fields: `bid_number`, `buyer_org`, `category`, `contract_period`, `emd_amount`, `epbg_percent`, `min_annual_turnover_lakhs`, etc.
- Uses `ask_freeform_question()` for each field (full RAG pipeline per query)
- Stored in `rfp_facts` table

#### d) Triage Chat
Users can ask free-form questions about the bid documents:
- **Canonical questions** (e.g., "What is the EMD amount?") are cached in `triage_signals`
- **Free-form questions** go directly to RAG + LLM, no caching
- Flow: question → ChromaDB retrieval (top-12 chunks) → AutoGen agent with grounded context → answer + citations

### 4.4 Stage 3: Proposal Generation

**Entry:** `POST /api/v1/generate/{bid_id}`  
The system has two generation pipelines:

#### V1 – Multi-Agent Enrichment Pipeline (`proposal_orchestrator.py`)

Uses **3 sequential writer-critic agent pairs**, each adding a layer of enrichment:

```
For each section:
  Pair 1: Bid Doc Writer + Critic
    └── Generates initial draft from RFP context + company knowledge
    └── 2 writer-critic iterations

  Pair 2: Cyberthreat Writer + Critic  (skipped for compliance/commercials)
    └── Enriches draft with live cyberthreat intelligence (Exa web search)
    └── 2 writer-critic iterations

  Pair 3: Tech Trends Writer + Critic  (skipped for compliance/commercials)
    └── Enriches draft with technology trends (Exa web search)
    └── 2 writer-critic iterations → final section content
```

**Enrichment agents** use the **Exa Search API** to find real-time web content:
- `search_cyberthreats()` — latest security threats, vulnerabilities, attack vectors
- `search_tech_trends()` — emerging cybersecurity technologies, AI/ML in security

This pipeline produces richer, more current content but takes longer (6 LLM calls per section × 2 iterations = 12+ LLM calls per section).

#### V2 – Subsection-Based Template Pipeline (`proposal_orchestrator_v2.py`)

Uses a **single writer agent** with **structured prompt templates** per subsection:

| Section | Subsections |
|---------|-------------|
| Executive Summary | Company overview, understanding of requirements, approach summary |
| Scope of Work | Service descriptions, SLA commitments, deliverables |
| Technical Solution | Architecture, deployment, monitoring, incident response |
| Compliance | Regulatory frameworks, certifications |
| Project Plan | Milestones, team structure, timelines |
| Commercials | Pricing (placeholder for manual fill) |

Each `SubSection` has:
- `output_type`: PROSE, BULLETS, or TABLE
- `prompt_template`: LLM prompt with `{variables}` for context injection
- `table_headers`: Column headers if output_type is TABLE

```
For each section:
  For each subsection:
    1. Build context:
       - RFP Facts (from triage)
       - Triage Requirements (filtered by relevance)
       - Company Knowledge (from blusapphire.json)
       - RAG chunks (bid-specific, retrieved from ChromaDB)
    2. Format prompt template with context variables
    3. Call AutoGen writer agent (1 LLM call per subsection)
    4. Save generated content to proposal.content JSON
```

This pipeline is faster and produces more template-consistent output (1 LLM call per subsection).

#### Context Sources for Generation

| Source | Provider | Description |
|--------|----------|-------------|
| RFP Facts | `rfp_facts_extractor.py` | Structured bid metadata (buyer, EMD, turnover, etc.) |
| Requirements | `triage_worker.py` | Extracted requirements with categories |
| Company Knowledge | `company_knowledge.py` | BluSapphire platform caps, services, compliance |
| RAG Chunks | `context_builder.py` | Semantically relevant chunks from bid documents |
| Web Search (V1 only) | `exa_tool.py` | Live cyberthreat and tech trend data |

### 4.5 Stage 4: Document Export

**Entry:** `GET /api/v1/export/{bid_id}/proposal`  
**Code:** `document_export.py`

Converts the proposal JSON into a formatted Word document:
- Each section gets a heading (numbered: 1.0, 1.1, etc.)
- Content formatted based on `output_type`:
  - **PROSE:** Paragraphs split on double newlines
  - **BULLETS:** Lines starting with `-` or `•` become bullet points
  - **TABLE:** Pipe-delimited data parsed into formatted Word tables with styled headers
- Cover page with bid title, BluSapphire branding
- Table of contents

---

## 5. PQ Auto-Fill Pipeline

The Pre-Qualification auto-fill system processes Excel questionnaires commonly used in government tenders.

### 5.1 End-to-End Flow

```
Upload Excel  →  Initialize KB  →  Setup AI Agent  →  Process Questions  →  Save Filled Excel
```

### 5.2 Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ PQ Knowledge │     │  PQ Worker   │     │ Filled Excel │
│   Base       │◀───▶│              │────▶│  (output)    │
│ (ChromaDB)   │     │  For each Q: │     │              │
│              │     │  1. KB query │     │ Color-coded  │
│ Q&A pairs    │     │  2. LLM call │     │ compliance   │
│ from past    │     │  3. Fill cell│     │ cells        │
│ RFPs         │     └──────────────┘     └──────────────┘
└──────────────┘
```

### 5.3 Knowledge Base Management

**Code:** `pq/vector_store.py`, `services/pq_kb_service.py`

The PQ knowledge base is a separate ChromaDB collection containing Q&A pairs from previously filled RFP questionnaires.

- **Ingestion:** Excel files with Question / Compliance / Remarks columns are ingested
- **Storage:** Each Q&A pair is embedded as a document in ChromaDB with metadata: `compliance`, `answer`, `source_sheet`, `doc_id`
- **Smart Column Detection:** `find_columns()` uses a hybrid approach:
  1. Keyword matching on headers (first 10 rows)
  2. Fallback to data-driven heuristics (longest avg text = question col, most Yes/No values = compliance col)

### 5.4 PQ Processing (`pq_worker.py`)

For each question in the input Excel:

```
1. [1/4] Initialize KB
   - Load ChromaDB PQ collection
   - Report total Q&A pairs available

2. [2/4] Setup AI Agent
   - Create AutoGen agent with structured output (RFPResponse model)
   - Model: xiaomi/mimo-v2-flash via OpenRouter

3. [3/4] Load Questionnaire
   - Parse input Excel, detect sheets
   - Auto-detect question/compliance/remarks columns per sheet

4. [4/4] Process Questions
   For each question cell:
     a. Semantic search → top 5 KB matches with similarity scores
     b. Format KB context (question, compliance, answer, match %)
     c. Single LLM call with system prompt:
        - "Analyze compliance and generate a direct capability remark"
        - Structured output: { compliance: "Yes"|"No"|"Partial", remarks: "..." }
     d. Write compliance + remarks to Excel cells
     e. Color-code: green(Yes), yellow(Partial), red(No)
```

### 5.5 LLM Prompt Design

The PQ system prompt enforces specific output rules:
- State WHAT the solution does, not WHETHER it complies
- No meta-commentary ("RFP requirement", "similarity", "match")
- No filler phrases ("fully compliant", "as stated")
- Focus on capability, not compliance language
- Short remarks (5-6 words) when KB has no detailed answer

---

## 6. Background Workers

Three async background workers run concurrently, started in the FastAPI lifespan handler:

| Worker | File | Job Model | Poll Interval |
|--------|------|-----------|---------------|
| Ingestion Worker | `jobs/ingestion_worker.py` | `IngestionJob` | 30s |
| Generation Worker | `jobs/generation_worker.py` | `GenerationJob` | 30s |
| PQ Worker | `jobs/pq_worker.py` | `PQJob` | 30s |

### 6.1 Job Lifecycle

```
pending → running → completed
                  → failed
                  → cancelled (PQ only, user-initiated)
```

### 6.2 Recovery & Reliability
- **Job recovery** on startup: any `running` jobs are reset to `pending` (`jobs/recovery.py`)
- **Heartbeat tracking:** `last_heartbeat` column updated during processing for stale job detection
- **Activity logging:** All workers write structured logs to `activity_logs` table for observability
- **Graceful shutdown:** Workers handle `CancelledError` on app shutdown

---

## 7. RAG (Retrieval-Augmented Generation)

### 7.1 Bid Documents RAG

| Component | File | Purpose |
|-----------|------|---------|
| Registry | `rag/chroma_registry.py` | Singleton ChromaDB client + embedding function |
| Lazy Proxy | `rag/chroma.py` | Deferred collection initialization for safe imports |
| Retrieval | `rag/retrieval.py` | `retrieve_chunks(bid_id, query, k)` |
| Context Builder | `rag/context_builder.py` | Section-specific retrieval profiles |
| Reader | `rag/chroma_reader.py` | `fetch_all_chunks()`, `fetch_relevant_chunks()` |

**Embedding model:** `baai/bge-m3` via OpenRouter API  
**Collection:** `bid_docs_v2` (cosine similarity, persistent)  
**Retrieval:** Top-k semantic search, filtered by `bid_id` metadata

### 7.2 PQ Knowledge Base

| Component | File | Purpose |
|-----------|------|---------|
| Vector Store | `pq/vector_store.py` | Singleton PQVectorStore class |
| KB Service | `services/pq_kb_service.py` | Document management (ingest/remove) |

**Embedding model:** `sentence-transformers/all-minilm-l6-v2` (local, via ChromaDB default)  
**Collection:** `pq_kb` (separate ChromaDB instance at `/data/pq_data/chroma_db`)  
**Search:** Top-5 matches with similarity percentage

---

## 8. AI Agents

Built on **Microsoft AutoGen** framework:

| Agent | File | Role |
|-------|------|------|
| Bid Doc Writer | `agents/bid_doc_agents.py` | Primary proposal writer (V1 + V2) |
| Bid Doc Critic | `agents/bid_doc_agents.py` | Reviews and suggests improvements (V1) |
| Cyberthreat Writer | `agents/enrichment_agents.py` | Enriches proposals with live threat intel (V1) |
| Cyberthreat Critic | `agents/enrichment_agents.py` | Reviews threat enrichment quality (V1) |
| Tech Trends Writer | `agents/enrichment_agents.py` | Enriches with emerging tech trends (V1) |
| Tech Trends Critic | `agents/enrichment_agents.py` | Final quality gate review (V1) |
| Triage Extractor | `agents/triage_extractor.py` | Extracts requirements from document chunks |
| Triage Chat | `agents/triage_chat_agent.py` | Answers questions about bid documents |
| PQ Agent | `jobs/pq_worker.py` | Structured compliance + remarks output |

**LLM Provider:** OpenRouter (`xiaomi/mimo-v2-flash`)  
**External Tools:** Exa Search API (cyberthreat + tech trends web search)  
**Structured Output:** Pydantic models for type-safe LLM responses (PQ worker)

---

## 9. API Endpoints

### 9.1 Bid Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/bids` | List all bids |
| POST | `/api/v1/bids` | Create a new bid |
| GET | `/api/v1/bids/{id}` | Get bid details + documents |

### 9.2 Document Operations
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/bids/{id}/documents/upload` | Upload and ingest a document |
| GET | `/api/v1/bids/{id}/documents` | List documents for a bid |
| GET | `/api/v1/documents/{id}/preview` | Preview document (inline PDF) |
| GET | `/api/v1/documents/{id}/download` | Download document |

### 9.3 Triage
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/triage/{bid_id}/run` | Start a triage run |
| POST | `/api/v1/triage/{bid_id}/ask` | Ask a triage question |
| POST | `/api/v1/triage/{bid_id}/chat` | Free-form document chat |
| GET | `/api/v1/triage/{bid_id}/signals` | Get triage signals |
| GET | `/api/v1/triage/{bid_id}/rfp-facts` | Get extracted RFP facts |

### 9.4 Proposal Generation
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/generate/{bid_id}` | Start proposal generation |
| GET | `/api/v1/generate/{bid_id}/progress` | Get generation progress |
| GET | `/api/v1/export/{bid_id}/proposal` | Export proposal as DOCX |

### 9.5 PQ Auto-Fill
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/pq/upload` | Upload PQ Excel and start processing |
| GET | `/api/v1/pq/jobs` | List PQ jobs |
| GET | `/api/v1/pq/jobs/{id}` | Get PQ job status + logs |
| GET | `/api/v1/pq/jobs/{id}/download` | Download filled Excel |
| POST | `/api/v1/pq/jobs/{id}/cancel` | Cancel a running PQ job |
| GET | `/api/v1/pq/kb/stats` | Get KB statistics |
| POST | `/api/v1/pq/kb/ingest` | Ingest a KB document |
| DELETE | `/api/v1/pq/kb/documents/{id}` | Remove a KB document |

### 9.6 System
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/health` | Health check |
| GET | `/api/v1/settings` | Get app settings |
| PUT | `/api/v1/settings` | Update app settings |
| GET | `/api/v1/activity-logs` | Query activity logs |
| GET | `/api/v1/activity-logs/sessions` | List job sessions |

---

## 10. Frontend Pages

| Page | Route | Key Features |
|------|-------|-------------|
| Dashboard | `/` | Bid list, quick stats, navigation |
| Add Bid | `/add-bid` | Create bid with title, department, dates |
| Bid Info | `/bid/:id` | Document list, upload, preview modal |
| Triage | `/bid/:id/triage` | RFP facts, triage questions, chat interface |
| Generate | `/bid/:id/generate` | Proposal generation with progress, section preview |
| PQ | `/pq` | Upload questionnaire, KB management, job monitoring |
| MyBrand | `/brand` | Company profile editor |
| Activity Log | `/activity-log` | Timeline + sessions view of all pipeline activity |
| Settings | `/settings` | API key, app configuration |

---

## 11. Deployment Architecture

### 11.1 Docker & Railway

```dockerfile
# Multi-stage build
FROM node:20-alpine AS frontend-build    # Build React frontend
FROM python:3.11-slim AS runtime         # Python runtime

# Persistent volume mounted at /data
ENV BIDGEN_DB_PATH=/data/dev.db
ENV BIDGEN_CHROMA_PATH=/data/chroma_data
ENV BIDGEN_UPLOADS_PATH=/data/uploads
ENV BIDGEN_PQ_DIR=/data/pq_data
ENV BIDGEN_PQ_CHROMA_PATH=/data/pq_data/chroma_db
```

### 11.2 Data Layout on Volume

```
/data/                         ← Railway persistent volume
├── dev.db                     ← SQLite database
├── chroma_data/bids/          ← ChromaDB for bid documents
├── uploads/{bid_id}/          ← Uploaded documents
├── pq_data/
│   ├── chroma_db/             ← ChromaDB for PQ knowledge base
│   ├── kb_documents/          ← PQ KB source Excel files
│   └── logs/                  ← PQ worker logs
├── logs/proposal_generation/  ← Proposal generation JSONL logs
└── settings.json              ← Application settings
```

### 11.3 First Deploy Seeding

`start.sh` handles first deploy:
1. Checks for `/data/.seeded` marker file
2. If not found: copies seed data (DB, ChromaDB, KB documents) from Docker image to volume
3. Creates marker file to prevent re-seeding on subsequent deploys

### 11.4 Desktop App (Tauri)

The same frontend + backend also runs as a desktop app using Tauri v2:
- Rust shell wraps the web UI
- Backend runs locally on `http://127.0.0.1:8000`
- All data stored in local `data/` directory

---

## 12. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Table-aware chunking** | Standard text chunking loses table context critical for government RFPs with specification tables |
| **Subsection-based generation (V2)** | Produces more structured, template-consistent output vs. free-form section generation (V1) |
| **Separate PQ knowledge base** | PQ questionnaires need cross-bid knowledge (past responses), unlike proposal generation which is bid-specific |
| **Deterministic KB + single LLM call (PQ)** | More reliable than multi-turn LLM conversations for compliance assessment; KB provides ground truth |
| **Lazy ChromaDB initialization** | Prevents import-time crashes when API keys aren't set; allows app to start regardless of LLM availability |
| **Activity Log system** | Cross-cutting observability for all background workers; essential for debugging deployed pipeline issues |
| **Non-fatal linked document collection** | Government sites (gem.gov.in) may be geo-blocked from cloud servers; main document ingestion must still succeed |

---

## 13. Security & Configuration

| Setting | Source | Notes |
|---------|--------|-------|
| `OPENROUTER_API_KEY` | Railway env var | Used for LLM calls and ChromaDB embeddings |
| `BIDGEN_*` paths | Dockerfile ENV | Points all data to persistent volume |
| CORS | `core/cors.py` | Configured for Railway + localhost origins |
| No authentication | By design | Internal tool, not public-facing |

---

## 14. File Structure Reference

```
Backend/
├── app/
│   ├── main.py                    # FastAPI app + worker startup
│   ├── agents/                    # AutoGen AI agents
│   │   ├── writer.py              # Proposal writer
│   │   ├── critic.py              # Proposal critic
│   │   ├── bid_doc_agents.py      # V2 subsection writer
│   │   ├── triage_extractor.py    # Requirement extractor
│   │   └── triage_chat_agent.py   # Document Q&A agent
│   ├── api/v1/                    # REST API endpoints
│   │   ├── router.py              # Route registration
│   │   ├── bids.py, documents.py  # Bid & document CRUD
│   │   ├── triage.py              # Triage endpoints
│   │   ├── generate.py            # Generation endpoints
│   │   ├── export.py              # DOCX export
│   │   ├── pq.py                  # PQ endpoints
│   │   ├── activity_log.py        # Activity log endpoints
│   │   └── settings.py, admin.py  # Configuration
│   ├── db/
│   │   ├── session.py             # SQLAlchemy async engine
│   │   ├── base.py                # Table creation
│   │   └── models/                # 15 SQLAlchemy models
│   ├── ingestion/
│   │   ├── parser.py              # Table-aware semantic chunking
│   │   └── doc_collector.py       # Linked document downloader
│   ├── jobs/
│   │   ├── ingestion_worker.py    # Document ingestion worker
│   │   ├── generation_worker.py   # Proposal generation worker
│   │   ├── pq_worker.py           # PQ auto-fill worker
│   │   ├── lock.py                # Job acquisition lock
│   │   └── recovery.py            # Stale job recovery
│   ├── pq/
│   │   └── vector_store.py        # PQ ChromaDB vector store
│   ├── rag/
│   │   ├── chroma_registry.py     # ChromaDB singleton (lazy)
│   │   ├── chroma.py              # Lazy collection proxy
│   │   ├── retrieval.py           # Semantic retrieval
│   │   ├── context_builder.py     # Section-specific RAG
│   │   └── chroma_reader.py       # Chunk reading utilities
│   ├── services/
│   │   ├── ingestion_service.py   # Ingestion orchestration
│   │   ├── proposal_orchestrator_v2.py  # V2 generation pipeline
│   │   ├── proposal_sections.py   # Section/subsection templates
│   │   ├── document_export.py     # DOCX rendering
│   │   ├── company_knowledge.py   # BluSapphire context
│   │   ├── rfp_facts_extractor.py # RFP metadata extraction
│   │   ├── triage_chat_service.py # RAG + LLM chat
│   │   ├── triage_worker.py       # Requirement extraction
│   │   ├── pq_service.py          # PQ job management
│   │   ├── pq_kb_service.py       # PQ KB management
│   │   ├── activity_log_service.py# Activity log service
│   │   └── proposal_logger.py     # Generation debug logs (JSONL)
│   └── storage/
│       └── local.py               # File upload storage
├── data/
│   └── company/blusapphire.json   # Company knowledge base
├── run_server.py                  # Server entry point
└── start.sh                       # Railway startup script

Frontend/
├── src/
│   ├── api/                       # API client functions
│   ├── components/                # Reusable UI components
│   ├── pages/                     # 10 page components
│   ├── hooks/                     # React hooks
│   └── types/                     # TypeScript type definitions
└── src-tauri/                     # Tauri desktop shell
```
