# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Build the Pinecone vector index (run once before starting the app, or when data/ changes)
python store_index.py

# Run the app
uvicorn app:app --host 0.0.0.0 --port 8080 --reload

# Docker build & run
docker build -t medibot .
docker run -d -p 8080:8080 --env-file .env medibot
```

## Environment Variables

Create a `.env` file in the root with:

```
PINECONE_API_KEY=your_key
GROQ_API_KEY=your_key
PINECONE_INDEX_NAME=medical-chatbot   # optional
GROQ_MODEL=llama-3.3-70b-versatile    # optional
RAG_TOP_K=5                           # optional
MAX_CHAT_HISTORY=10                   # optional
MAX_TOKENS=1024                       # optional
DEBUG=true                            # optional
```

## Architecture

Two distinct phases that must run in order:

**1. Indexing phase (`store_index.py`)** — one-time data pipeline:
- Loads PDFs from `data/` via `src/helper.py:load_pdf_file`
- Cleans, sentence-splits, and deduplicates text chunks (800-char chunks, 150-char overlap)
- Encodes chunks using `sentence-transformers/all-MiniLM-L6-v2` → 384-dim vectors
- Creates Pinecone serverless index (`medical-chatbot`, cosine, AWS us-east-1) if not present
- Upserts vectors in batches of 100; metadata includes `text`, `source`, `page`

**2. Serving phase (`app.py`)** — FastAPI app:
- All heavy clients (embedding model, Pinecone index, Groq client) are **lazily initialized** on first request, not at startup
- Per-session chat history stored in-memory as `dict[str, deque]` — resets on restart
- Chat flow: embed query → query Pinecone (filter score > 0.3) → inject context into system prompt → call Groq LLaMA 3.3 → return JSON with `answer`, `sources`, `elapsed`
- System prompt lives in `src/prompt.py`; the `{context}` placeholder is replaced with retrieved passages at runtime

**Frontend** (`static/templates/chat.html` + `static/style.css`):
- Vanilla JS SPA served by Jinja2
- POSTs to `/get` with `msg` and `session_id` form fields
- Renders markdown-like formatting from the LLM response

## Key Constraints

- `store_index.py` must be run before `app.py` — the app will fail at query time if the Pinecone index is empty
- Pinecone index dimension is hardcoded to `384` (matches `all-MiniLM-L6-v2`); changing the embedding model requires re-indexing
- Session history is in-memory only — horizontal scaling requires a shared store (Redis, etc.)
- `CORS` is open (`allow_origins=["*"]`) — lock this down for production
