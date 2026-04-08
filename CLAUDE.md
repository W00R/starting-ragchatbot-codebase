# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies (always use uv, never pip)
uv sync

# Start the dev server (from repo root)
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# App: http://localhost:8000
# API docs: http://localhost:8000/docs
```

No test framework or linter is configured.

## Environment

Requires Python 3.13+, uv, and an `ANTHROPIC_API_KEY` in a root `.env` file (see `.env.example`). On Windows, use Git Bash to run shell commands.

## Architecture

A RAG (Retrieval-Augmented Generation) chatbot for course materials. FastAPI backend serves a vanilla HTML/JS frontend and exposes two API endpoints: `POST /api/query` and `GET /api/courses`.

### Query flow

1. Frontend POSTs `{query, session_id}` to `/api/query`
2. `app.py` routes to `RAGSystem.query()` which wraps the prompt and retrieves conversation history
3. `AIGenerator` calls Claude API (`claude-sonnet-4`) **with tools enabled** (`tool_choice: auto`)
4. Claude decides whether to call `search_course_content` tool or answer directly from general knowledge
5. If tool is called: `CourseSearchTool` → `VectorStore.search()` → ChromaDB semantic search → results returned as `tool_result` → Claude called **a second time** (without tools) to synthesize the final answer
6. Response + sources returned to frontend, rendered as Markdown via `marked.js`

### Key design decisions

- **Two ChromaDB collections**: `course_catalog` (course metadata for semantic name resolution) and `course_content` (text chunks for content retrieval)
- **Tool-based search**: Claude autonomously decides whether to search via Anthropic tool use, rather than always retrieving
- **Session history**: In-memory only (`SessionManager`), capped at 2 exchanges, injected into system prompt as plain text
- **Document format**: Files in `docs/` must follow a strict header format — lines 1-3: `Course Title:`, `Course Link:`, `Course Instructor:`, then `Lesson N: Title` markers with optional `Lesson Link:` lines. See `DocumentProcessor.process_course_document()`
- **Chunking**: Sentence-based splitting (800 char chunks, 100 char overlap) with contextual prefixes like `"Lesson N content: ..."`
- **Startup loading**: `app.py` startup event auto-loads all `.txt/.pdf/.docx` from `docs/`, skipping courses already in ChromaDB

### Backend modules (`backend/`)

- `app.py` — FastAPI app, routes, static file serving
- `rag_system.py` — Orchestrator wiring all components
- `ai_generator.py` — Claude API calls with single-round tool use loop
- `vector_store.py` — ChromaDB wrapper with course name resolution and filtered search
- `document_processor.py` — File parsing, metadata extraction, sentence-based chunking
- `search_tools.py` — Tool abstraction (`Tool` ABC, `ToolManager`, `CourseSearchTool`)
- `session_manager.py` — In-memory conversation history per session
- `models.py` — Pydantic/dataclass models: `Course`, `Lesson`, `CourseChunk`
- `config.py` — Config dataclass loaded from env vars

### Frontend

Vanilla HTML/CSS/JS in `frontend/`, served as static files at `/` by FastAPI. Uses `marked.js` from CDN for Markdown rendering. No build step.
