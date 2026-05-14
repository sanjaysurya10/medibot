# Product Requirements Document
## MediBot — Clinical Decision Support System (RAG-Powered Medical Chatbot)

**Version:** 2.0.0
**Date:** 2026-05-14
**Status:** In Development

---

## 1. Overview

MediBot is a RAG-powered clinical decision support chatbot that answers medical queries by grounding every response in a curated knowledge base of medical literature. It targets healthcare professionals, medical students, and informed patients who need fast, evidence-based answers without waiting for a clinician.

**Core value proposition:** No hallucinations from general LLM knowledge — every answer is anchored to retrieved source documents with citations, while LLaMA 3.3 70B provides fluent, clinically structured prose.

---

## 2. Problem Statement

General-purpose LLMs hallucinate medical facts with dangerous confidence. Clinicians and students cannot trust a chatbot that makes up dosages, contraindications, or diagnostic criteria. Existing solutions either require expensive fine-tuning, lack source attribution, or provide no safety guardrails.

MediBot solves this by combining a curated vector knowledge base (Pinecone) with a low-temperature LLM call (Groq/LLaMA) and a clinically-designed system prompt that forces grounding, emergency escalation, and source transparency.

---

## 3. Goals

| Goal | Metric |
|------|--------|
| Reduce hallucination risk | >90% of answers cite a retrieved source (score > 0.3) |
| Response latency | <5 seconds per query |
| Safety compliance | 100% of emergency-symptom queries escalate to emergency services |
| Source transparency | Every response includes source filename + page number |
| Availability | Runs locally and in Docker with a single command |

---

## 4. Non-Goals

- Real-time EHR integration
- User authentication / patient record storage
- Fine-tuned model training
- Regulatory medical device certification (FDA/CE)
- Streaming token-by-token responses (currently returns full response)

---

## 5. Users

| Persona | Use Case |
|---------|----------|
| **Medical Student** | Quick differential diagnosis lookup, drug mechanism review |
| **Healthcare Professional** | Fast clinical guideline reference during rounds |
| **Informed Patient** | Understand a diagnosis or medication before a consultation |
| **Developer/Researcher** | Extend the RAG pipeline with new data sources |

---

## 6. Functional Requirements

### 6.1 Data Ingestion (`store_index.py`)
- **FR-01:** Load all PDF files recursively from the `data/` directory
- **FR-02:** Clean OCR artifacts and normalize whitespace before chunking
- **FR-03:** Sentence-aware chunking with configurable size (default 800 chars) and overlap (default 150 chars)
- **FR-04:** Deduplicate chunks by MD5 content hash before indexing
- **FR-05:** Auto-create Pinecone serverless index if it doesn't exist (384-dim, cosine, AWS us-east-1)
- **FR-06:** Batch upsert vectors in groups of 100 with error handling per batch

### 6.2 Chat API (`app.py`)
- **FR-07:** `POST /get` accepts `msg` (text) and `session_id` (string) as form fields
- **FR-08:** Embed user query using `all-MiniLM-L6-v2` and retrieve top-K chunks from Pinecone
- **FR-09:** Filter retrieved chunks to those with similarity score > 0.3
- **FR-10:** Inject retrieved context into system prompt `{context}` placeholder
- **FR-11:** Maintain per-session conversation history (deque, max 10 message pairs) for multi-turn context
- **FR-12:** Call Groq LLaMA 3.3 70B with temperature=0.3, top_p=0.9, max_tokens=1024
- **FR-13:** Return JSON with `answer`, `sources` (list of `{source, page, score}`), `elapsed`, `timestamp`
- **FR-14:** `POST /clear-history` clears the session's conversation history
- **FR-15:** `GET /health` returns vector count, model name, and service status

### 6.3 Safety & Clinical Protocols
- **FR-16:** System prompt must always direct chest pain / stroke / severe allergic reactions to emergency services before any other content
- **FR-17:** Every response must end with a disclaimer that it doesn't replace professional medical advice
- **FR-18:** Never recommend specific dosages for controlled substances

### 6.4 Frontend (`static/templates/chat.html`)
- **FR-19:** Single-page chat UI with auto-scrolling message area
- **FR-20:** Display source citations (filename + page + relevance score) beneath each response
- **FR-21:** Show elapsed response time per message
- **FR-22:** Quick-topic buttons for common medical query categories
- **FR-23:** Character counter with 2000-char limit on input
- **FR-24:** Copy-to-clipboard button per assistant message

---

## 7. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| **Latency** | p95 < 5s end-to-end (embedding + retrieval + LLM) |
| **Embedding model** | `all-MiniLM-L6-v2` — 384 dims, runs CPU-only |
| **LLM** | Groq-hosted LLaMA 3.3 70B (not self-hosted) |
| **Vector store** | Pinecone Serverless (no infrastructure to manage) |
| **Session storage** | In-memory (no persistence across restarts) |
| **Containerization** | Single-stage Dockerfile, Python 3.12-slim base |
| **CORS** | Open in dev; must be restricted to known origins in production |

---

## 8. Architecture Summary

```
User Query
    │
    ▼
FastAPI /get endpoint (app.py)
    │
    ├─► Embed query (all-MiniLM-L6-v2, 384-dim)
    │
    ├─► Query Pinecone → top-5 chunks (score > 0.3)
    │
    ├─► Build messages:
    │     [system_prompt with {context}] + [session history] + [user msg]
    │
    ├─► Groq API → LLaMA 3.3 70B (temp=0.3)
    │
    └─► Return {answer, sources, elapsed}
```

**Data pipeline (one-time):**
```
data/*.pdf → load_pdf_file → clean_text → text_split (sentence-aware)
    → batch_encode (all-MiniLM-L6-v2) → Pinecone upsert
```

---

## 9. API Reference

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Chat UI (HTML) |
| `POST` | `/get` | Chat endpoint (`msg`, `session_id` form fields) |
| `POST` | `/clear-history` | Clear session history (`session_id` form field) |
| `GET` | `/health` | Health + Pinecone vector count |
| `GET` | `/api/test-groq` | Test Groq API connectivity |
| `GET` | `/api/info` | API metadata |

---

## 10. Configuration Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `PINECONE_API_KEY` | required | Pinecone credentials |
| `GROQ_API_KEY` | required | Groq LLM credentials |
| `PINECONE_INDEX_NAME` | `medical-chatbot` | Pinecone index name |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | LLM model ID |
| `RAG_TOP_K` | `5` | Retrieved chunks per query |
| `MAX_CHAT_HISTORY` | `10` | Conversation history depth (message pairs) |
| `MAX_TOKENS` | `1024` | Max LLM response tokens |
| `DEBUG` | `true` | Expose error details in API responses |
| `PORT` | `8080` | HTTP server port |

---

## 11. Open Issues & Future Work

| Priority | Item |
|----------|------|
| High | Add more medical PDFs to `data/` — current knowledge base is a single textbook |
| High | Lock down CORS to specific origins before any public deployment |
| Medium | Streaming responses (SSE or WebSocket) for better perceived latency |
| Medium | Persistent session storage (Redis) for horizontal scaling |
| Medium | Auth layer (API key or OAuth) before exposing publicly |
| Low | Re-ranking retrieved chunks (cross-encoder) for better context quality |
| Low | Evaluation harness — measure retrieval recall and answer faithfulness |
| Low | Expand `store_index.py` to support web scraping and FHIR data sources |

---

## 12. Known Constraints

- Pinecone index dimension is hardcoded to `384` — changing the embedding model requires full re-indexing
- Session history is lost on app restart (in-memory only)
- `store_index.py` must complete successfully before the app can answer any queries
- Python 3.12+ required (uses `dict[str, deque]` type hint syntax)
