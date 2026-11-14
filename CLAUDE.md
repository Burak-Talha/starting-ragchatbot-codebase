# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A RAG chatbot that answers questions about course transcripts in `docs/`. FastAPI backend (`backend/`), ChromaDB vector store, Anthropic Claude for generation, and a vanilla HTML/JS frontend (`frontend/`) served by the same FastAPI app.

## Commands

Uses `uv` (Python ≥3.13). There is no test suite, linter, or build step configured.

**Always use `uv` to run the server and manage dependencies — never use `pip` directly** (e.g. `uv add <pkg>`, not `pip install`; `uv run ...`, not bare `python`/`uvicorn`).

```bash
uv sync                      # install dependencies
cp .env.example .env         # then set ANTHROPIC_API_KEY
./run.sh                     # start the server on http://localhost:8000 (API docs at /docs)

# manual start — must be run from backend/
cd backend && uv run uvicorn app:app --reload --port 8000
```

The server **must run with `backend/` as the working directory**: `app.py` uses relative paths `../docs`, `../frontend`, and `./chroma_db` (config `CHROMA_PATH`). The root `main.py` is an unused placeholder.

To force re-ingestion of documents, delete `backend/chroma_db/`. Startup ingestion skips any course whose title already exists in ChromaDB, so edits to an existing course file are not picked up otherwise.

## Architecture

### Ingestion (runs on FastAPI startup)
`app.py` startup hook → `RAGSystem.add_course_folder("../docs")` → `DocumentProcessor.process_course_document()` → `VectorStore.add_course_metadata()` + `add_course_content()`.

- Course files must follow a header format parsed by regex in `document_processor.py`: line 1 `Course Title:`, then `Course Link:` / `Course Instructor:`, then `Lesson N: <title>` markers each optionally followed by a `Lesson Link:` line. The **course title is the unique ID** throughout (Chroma IDs, filters, dedup).
- Chunking is sentence-based (`CHUNK_SIZE`=800 chars, `CHUNK_OVERLAP`=100 chars of trailing sentences). Chunks get a `Lesson N content:` prefix — note the last lesson uses a different prefix (`Course <title> Lesson N content:` on every chunk) than earlier lessons (prefix on the first chunk only).
- Only `.txt` is really supported; `.pdf`/`.docx` pass the extension filter but are read as plain text.

ChromaDB has **two collections**, both embedded with `all-MiniLM-L6-v2`:
- `course_catalog` — one doc per course (title text); metadata holds instructor, links, and `lessons_json` (serialized lesson list). Used to resolve fuzzy course names.
- `course_content` — the chunks, with `course_title` / `lesson_number` / `chunk_index` metadata for filtering.

### Query path (tool-based RAG)
`POST /api/query` → `RAGSystem.query()` → `AIGenerator.generate_response()`.

Retrieval is **decided by Claude via tool use**, not done unconditionally:
1. First Claude call gets the `search_course_content` tool (`tool_choice: auto`). Conversation history is injected as text into the **system prompt**, not as message turns.
2. If Claude returns `tool_use`, `ToolManager` dispatches to `CourseSearchTool.execute()` → `VectorStore.search()`, which resolves `course_name` via semantic search on `course_catalog` (always returns the nearest match), builds a Chroma `where` filter, and queries `course_content` for `MAX_RESULTS` chunks.
3. A second Claude call is made with the tool result and **no tools**, so at most one search happens per query. The system prompt in `ai_generator.py` also instructs "one search per query maximum".

Sources for the UI flow through a side channel: `CourseSearchTool._format_results` sets `self.last_sources`, and `RAGSystem.query` reads then resets them via `ToolManager.get_last_sources()` / `reset_sources()`. This is shared mutable state on a single tool instance.

New tools: subclass `Tool` in `search_tools.py` (implement `get_tool_definition()` returning an Anthropic tool schema and `execute(**kwargs)`), then register it with `tool_manager` in `RAGSystem.__init__`.

### Sessions
`SessionManager` is in-memory only (lost on restart). IDs are `session_N`; it keeps the last `MAX_HISTORY * 2` messages. The frontend gets the `session_id` from the first response and sends it back on later queries.

### Config
All tunables (model name, embedding model, chunk sizes, `MAX_RESULTS`, `MAX_HISTORY`, Chroma path) live in the `Config` dataclass in `backend/config.py`; `ANTHROPIC_API_KEY` comes from `.env`.

### Frontend
`frontend/script.js` calls `/api/query` and `/api/courses`, renders assistant replies as Markdown via `marked` (loaded from CDN in `index.html`), and shows sources in a collapsible `<details>`. It's served by `StaticFiles` mounted at `/` in `app.py` — that mount must stay last so it doesn't shadow `/api/*` routes.
