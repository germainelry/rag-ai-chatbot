# System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PUBLIC INTERNET                                │
│                                                                             │
│   ┌──────────────────────┐          ┌──────────────────────────────────┐    │
│   │   Customer Browser   │          │      Agent / Admin Browser       │    │
│   │                      │          │                                  │    │
│   │  ┌────────────────┐  │          │  ┌────────────────────────────┐  │    │
│   │  │  Chat Widget   │  │          │  │     Agent Dashboard SPA    │  │    │
│   │  │  (React + WS)  │  │          │  │  (React 18 + TypeScript)   │  │    │
│   │  └───────┬────────┘  │          │  └────────────┬───────────────┘  │    │
│   └──────────┼───────────┘          └───────────────┼──────────────────┘    │
│              │ WebSocket / HTTPS                    │ HTTPS / REST          │
└──────────────┼──────────────────────────────────────┼───────────────────────┘
               │                                      │
┌──────────────▼──────────────────────────────────────▼────────────────────────┐
│                         BACKEND PLATFORM (Railway)                           │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                       FastAPI Application Server                       │  │
│  │                                                                        │  │
│  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐     │  │
│  │   │  Auth Layer  │  │  Rate Limit  │  │    WebSocket Hub         │     │  │
│  │   │  (JWT/OAuth) │  │  (SlowAPI)   │  │  (Real-time messaging)   │     │  │
│  │   └──────────────┘  └──────────────┘  └──────────────────────────┘     │  │
│  │                                                                        │  │
│  │   ┌──────────────────────────────────────────────────────────────┐     │  │
│  │   │                    Core Business Logic                       │     │  │
│  │   │                                                              │     │  │
│  │   │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐    │     │  │
│  │   │  │   Routing   │  │  HITL Flow   │  │  Knowledge Base   │    │     │  │
│  │   │  │   Engine    │  │  Controller  │  │  Ingest Pipeline  │    │     │  │
│  │   │  └─────────────┘  └──────┬───────┘  └─────────┬─────────┘    │     │  │
│  │   │                          │                    │              │     │  │
│  │   └──────────────────────────┼────────────────────┼──────────────┘     │  │
│  │                              │                    │                    │  │
│  │   ┌───────────────────────── ▼ ────────────────── ▼ ───────────────┐   │  │
│  │   │                     AI / ML Layer                              │   │  │
│  │   │                                                                │   │  │
│  │   │  ┌─────────────────────────┐  ┌───────────────────────────┐    │   │  │
│  │   │  │      RAG Engine         │  │    LLM Inference Layer    │    │   │  │
│  │   │  │                         │  │                           │    │   │  │
│  │   │  │  1. Embed query         │  │  • Prompt construction    │    │   │  │
│  │   │  │  2. Similarity search   │──►│  • Context injection     │    │   │  │
│  │   │  │  3. Retrieve top-k      │  │  • Draft generation       │    │   │  │
│  │   │  │     chunks              │  │  • Confidence scoring     │    │   │  │
│  │   │  └────────────┬────────────┘  └─────────────┬─────────────┘    │   │  │
│  │   │               │                             │                  │   │  │
│  │   └───────────────┼─────────────────────────────┼──────────────────┘   │  │
│  │                   │                             │                      │  │
│  └───────────────────┼─────────────────────────────┼──────────────────────┘  │
│                      │                             │                         │
└──────────────────────┼─────────────────────────────┼─────────────────────────┘
                       │                             │
         ┌─────────────▼───────┐        ┌────────────▼─────────────────┐
         │     ChromaDB        │        │  HuggingFace Inference API   │
         │  (Vector Store)     │        │  (Serverless LLM — external) │
         │  Local to backend   │        │  No GPU required             │
         └─────────────────────┘        └──────────────────────────────┘

         ┌─────────────────────┐        ┌──────────────────────────────┐
         │  PostgreSQL         │        │   Supabase Storage           │
         │  (Supabase managed) │        │   (Document file storage)    │
         │  Users, messages,   │        │   PDFs, DOCX, CSVs           │
         │  conversations,     │        │                              │
         │  feedback, A/B data │        │                              │
         └─────────────────────┘        └──────────────────────────────┘
```

---

## Request Lifecycle — Typical HITL Flow

```
Step 1 — Customer sends message
  Customer browser → WebSocket → FastAPI WebSocket Hub
  Message persisted to PostgreSQL (conversations + messages tables)

Step 2 — AI Draft Generation triggered
  HITL Flow Controller pulls conversation context
  RAG Engine:
    a. Embeds the customer query
    b. Searches ChromaDB for top-k relevant knowledge chunks
    c. Returns ranked chunks above relevance threshold
  LLM Layer:
    a. Constructs prompt (system template + retrieved chunks + conversation history)
    b. Calls HuggingFace Inference API
    c. Receives generated draft text
    d. Scores confidence
  Draft + confidence score persisted to PostgreSQL

Step 3 — Agent is notified
  Agent dashboard receives real-time notification (WebSocket broadcast)
  Draft appears in agent's review queue

Step 4 — Agent reviews and acts
  Option A — Approve: draft sent as-is to customer
  Option B — Edit then send: agent modifies draft, sends edited version
  Option C — Reject: agent discards draft, writes manual response
  Decision logged to feedback table (used for future model improvement)

Step 5 — Response delivered to customer
  Approved/edited message pushed via WebSocket to customer session
  Conversation state updated in PostgreSQL
```

---

## Knowledge Base Ingestion Pipeline

```
Admin uploads document (PDF / DOCX / CSV)
  │
  ▼
Validation layer
  • MIME type check
  • File size limit enforced
  • Malformed file detection
  │
  ▼
Document Parser
  • PDF: page-by-page text extraction
  • DOCX: paragraph and table extraction
  • CSV: row-by-row field extraction
  │
  ▼
Chunker
  • Split into overlapping text chunks
  • Configurable chunk size and overlap
  │
  ▼
Embedding Model
  • Each chunk converted to a dense vector
  │
  ▼
ChromaDB
  • Vectors + raw chunk text stored
  • Immediately available for semantic search
  │
  ▼
Status update returned to Admin UI
```

---

## Authentication & Authorisation Flow

| Step | Detail |
|---|---|
| Credentials submitted | Email/password (bcrypt) or OAuth token |
| JWT issued | Short-lived access token + refresh token |
| Subsequent requests | Bearer token in Authorization header; middleware verifies signature + expiry |
| Role enforcement | `customer` → chat only · `agent` → dashboard + conversations · `admin` → all + user/KB management |

---

## Deployment Topology

```
┌───────────────────────────────────────────────────────┐
│                     Railway Platform                  │
│                                                       │
│   ┌──────────────────────┐   ┌────────────────────┐   │
│   │   Backend Service    │   │  Frontend Service  │   │
│   │   (Docker container) │   │  (Static build)    │   │
│   │   FastAPI + uvicorn  │   │  React SPA         │   │
│   │   ~50MB base image   │   │                    │   │
│   └──────────┬───────────┘   └────────────────────┘   │
│              │                                        │
└──────────────┼────────────────────────────────────────┘
               │
    ┌──────────┼──────────────────────────┐
    │          │       Supabase           │
    │   ┌──────▼────────┐  ┌───────────┐  │
    │   │  PostgreSQL   │  │  Storage  │  │
    │   │  (managed DB) │  │  (files)  │  │
    │   └───────────────┘  └───────────┘  │
    └─────────────────────────────────────┘
```

**Resource constraints addressed:**
- Entire backend operates within 0.5 GB RAM
- No local GPU or model weights — all LLM inference is API-based
- ChromaDB runs in-process (no separate vector DB service needed)
- PostgreSQL connection pooling limits concurrent connections

---

## Data Flow Summary

| Data Type | Storage | Access Pattern |
|---|---|---|
| User accounts | PostgreSQL | Read on auth, write on register |
| Conversations & messages | PostgreSQL | Read/write per WebSocket session |
| AI drafts & feedback | PostgreSQL | Written per HITL cycle; read for analytics |
| Knowledge base documents | Supabase Storage | Write on upload; read-only after ingestion |
| Knowledge base vectors | ChromaDB | Write on ingest; read on every LLM request |
| JWT tokens | Client-side (httpOnly or memory) | Sent with every API request |
| Session state | In-memory (WebSocket registry) | Per-connection lifetime |
