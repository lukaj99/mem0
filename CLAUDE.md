# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mem0 is an intelligent memory layer for AI assistants and agents. The repo contains three main components:

1. **`mem0/`** — The core Python SDK (`pip install mem0ai`), published to PyPI
2. **`openmemory/`** — A self-hosted MCP server + dashboard (FastAPI backend + Next.js UI), run via Docker Compose
3. **`embedchain/`** — A legacy RAG framework (separate package, largely independent)

## Build & Test Commands

### Core SDK (`mem0/`)
```bash
make install          # Create hatch env
make test             # Run tests via hatch (pytest tests/)
make lint             # Lint with ruff
make format           # Format with ruff
hatch run test -- tests/llms/test_openai.py        # Single test file
hatch run test -- tests/llms/test_openai.py -k test_name  # Single test
```

### OpenMemory (`openmemory/`)
```bash
cd openmemory
make env              # Copy .env.example files for api/ and ui/
make build            # docker compose build
make up               # docker compose up (Qdrant + API on :8765 + UI on :3012)
make down             # docker compose down -v + remove sqlite db
make migrate          # alembic upgrade head (inside container)
make ui-dev           # pnpm install && pnpm dev for the UI
```

OpenMemory API runs on port 8765, UI on port 3012 (mapped from 3000 inside container). Qdrant on port 6333.

## Architecture

### Core SDK (`mem0/`)

The `Memory` class (`mem0/memory/main.py`) is the central entry point. It orchestrates:
- **LLM** — extracts facts from conversations using prompts in `mem0/configs/prompts.py`
- **Embeddings** — vectorizes memories for similarity search
- **Vector Store** — stores and retrieves memory vectors (default: Qdrant)
- **SQLite** — tracks memory history/changelog (`mem0/memory/storage.py`)
- **Graph Store** — optional knowledge graph via Neo4j/Memgraph/Kuzu (`mem0/memory/graph_memory.py`)
- **Reranker** — optional result reranking

All providers are pluggable via factory pattern (`mem0/utils/factory.py`). Each factory maps provider names to lazy-loaded classes. To add a new provider: add the class in the appropriate directory, register it in the factory, and add a config class.

**Two usage modes:**
- `Memory` — local/self-hosted, uses configured LLM + vector store directly
- `MemoryClient` / `AsyncMemoryClient` — API client for the hosted Mem0 platform (api.mem0.ai)

Both are exported from `mem0/__init__.py`.

Configuration uses Pydantic models rooted at `MemoryConfig` (`mem0/configs/base.py`), composing `LlmConfig`, `EmbedderConfig`, `VectorStoreConfig`, `GraphStoreConfig`, and `RerankerConfig`.

The `Mem0` proxy class (`mem0/proxy/main.py`) wraps litellm to provide a drop-in OpenAI-compatible interface with automatic memory.

### OpenMemory (`openmemory/`)

A self-hosted personal memory server with MCP (Model Context Protocol) support:

- **`api/`** — FastAPI app (`main.py`). SQLAlchemy models in `app/models.py` with SQLite default (`app/database.py`). Alembic for migrations. MCP server in `app/mcp_server.py` using `mcp.server.fastmcp.FastMCP` with SSE transport.
- **`ui/`** — Next.js 15 + React 19 app with Redux Toolkit, Tailwind, shadcn/ui components. Uses pnpm.

Key models: `User`, `App`, `Memory`, `Category`, `AccessControl`, `ArchivePolicy`, `MemoryStatusHistory`, `MemoryAccessLog`. Memories are auto-categorized via LLM on insert/update (SQLAlchemy event listeners).

Docker Compose runs three services: `mem0_store` (Qdrant), `openmemory-mcp` (FastAPI), `openmemory-ui` (Next.js).

## Key Conventions

- Build system: **Hatch** (hatchling backend, hatch CLI for envs/scripts)
- Linting/formatting: **ruff** (line-length 120, excludes `embedchain/` and `openmemory/`)
- Testing: **pytest** with pytest-mock and pytest-asyncio. Tests mirror source structure under `tests/`.
- Python: 3.9+ required, type hints used, Pydantic v2 for config models
- API responses use `{"results": [...]}` format (v1.0+ change)
- `MEM0_DIR` env var overrides default `~/.mem0` storage directory
