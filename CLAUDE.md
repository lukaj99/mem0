# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mem0 is an intelligent memory layer for AI assistants and agents. Four main components:

1. **`mem0/`** — Core Python SDK (`pip install mem0ai`), published to PyPI
2. **`openmemory/`** — Self-hosted MCP server + dashboard (FastAPI + Next.js + Qdrant), run via Docker Compose
3. **`synaptic_memory/`** — Citation tracking, strength dynamics, and network centrality layer that augments Mem0 memories with inter-memory relationships
4. **`embedchain/`** — Legacy RAG framework (separate package, largely independent — ignore unless specifically asked)

## Build & Test Commands

### Core SDK (`mem0/`)
```bash
# This machine has a .venv (uv/cpython-3.12), not hatch CLI.
# Use .venv/bin/python -m pytest directly, or activate the venv.
.venv/bin/python -m pytest tests/llms/test_openai.py           # Single test file
.venv/bin/python -m pytest tests/llms/test_openai.py -k test_name  # Single test

# Some test files fail to import due to missing optional deps (azure, gemini, etc).
# Use --ignore or target specific directories rather than running all of tests/.
.venv/bin/python -m pytest tests/llms/ tests/vector_stores/ -x  # Targeted test run

# Linting (ruff config: line-length 120, excludes embedchain/ and openmemory/)
.venv/bin/python -m ruff check mem0/
.venv/bin/python -m ruff format mem0/
```

### Synaptic Memory (`synaptic_memory/`)
```bash
# All tests are async (pytest-asyncio). Fixtures use tmp_path for isolated SQLite DBs.
.venv/bin/python -m pytest tests/synaptic/ -x                  # All synaptic tests
.venv/bin/python -m pytest tests/synaptic/test_strength.py -k test_name  # Single test

# CLI (argparse-based, runs via asyncio.run)
python -m synaptic_memory.main stats                            # System-wide stats
python -m synaptic_memory.main add <memory_id> "<content>"      # Register memory
python -m synaptic_memory.main network <memory_id>              # ASCII network graph
python -m synaptic_memory.main decay                            # Run decay cycle
python -m synaptic_memory.main pagerank                         # Recalculate centrality
python -m synaptic_memory.main replay                           # Check due reviews
python -m synaptic_memory.main report --top 10                  # Top synapses by strength
```

### OpenMemory (`openmemory/`)
```bash
cd openmemory
make build            # docker compose build
make up               # docker compose up (Qdrant :6333 + API :8765 + UI :3012)
make down             # docker compose down -v + remove sqlite db
make migrate          # alembic upgrade head (inside container)
make ui-dev           # pnpm install && pnpm dev for the UI
```

Rebuild just the API after code changes (volumes mount `api/` so uvicorn --reload handles most changes, but env/deps changes need rebuild):
```bash
cd openmemory && docker compose up -d --build openmemory-mcp
```

## Architecture

### Core SDK (`mem0/`)

**`Memory`** class (`mem0/memory/main.py`) is the central entry point. Parallel **`AsyncMemory`** class in the same file. Both orchestrate:
- **LLM** → extracts facts from conversations (prompts in `mem0/configs/prompts.py`)
- **Embeddings** → vectorizes memories for similarity search
- **Vector Store** → stores/retrieves memory vectors (default: Qdrant)
- **SQLite** → tracks memory history/changelog (`mem0/memory/storage.py`)
- **Graph Store** → optional knowledge graph via Neo4j/Memgraph/Kuzu (`mem0/memory/graph_memory.py`)
- **Reranker** → optional result reranking

**Data flow for `memory.add()`:**
1. `_build_filters_and_metadata()` (module-level helper) resolves user_id/agent_id/run_id into metadata + query filters
2. Messages parsed → sent to LLM for fact extraction
3. Facts embedded → searched against existing memories for deduplication
4. LLM decides ADD/UPDATE/DELETE/NOOP for each fact
5. Vector store + optional graph store updated
6. SQLite history recorded

**Provider system:** All providers are pluggable via factory pattern (`mem0/utils/factory.py`). Each factory maps provider names to lazy-loaded classes. To add a new provider: add the class in the appropriate directory, register it in the factory, and add a config class.

**Configuration:** Pydantic models rooted at `MemoryConfig` (`mem0/configs/base.py`), composing `LlmConfig`, `EmbedderConfig`, `VectorStoreConfig`, `GraphStoreConfig`, `RerankerConfig`.

**Single-user default:** `MemoryConfig.default_user_id` (or env `MEM0_DEFAULT_USER_ID`) eliminates the need to pass `user_id` on every call. All public methods (`add`, `search`, `get_all`, `delete_all`) fall back to this.

**Two client modes:**
- `Memory` / `AsyncMemory` — local/self-hosted, uses configured LLM + vector store directly
- `MemoryClient` / `AsyncMemoryClient` — API client for hosted Mem0 platform (api.mem0.ai)

### Synaptic Memory (`synaptic_memory/`)

A neuroscience-inspired layer that tracks **relationships between memories** (synapses) and computes network-level metrics. It operates alongside Mem0's vector store, adding citation tracking, strength dynamics, and memory consolidation.

**`SynapticMemorySystem`** (`system.py`) is the main integration class. Async context manager, uses two SQLite databases:
- **`SynapseDB`** (`synapse_db.py`) — stores `Synapse` records (directed edges between memory IDs) with strength, citation type, access counts, temporal bias, and co-citation counts
- **`MemoryMetricsDB`** (`memory_db.py`) — stores per-memory aggregate metrics (`MemoryAugmented`: PageRank, hub score, importance, replay state) plus a `replay_queue` table

**Core data models** (`models.py`, all dataclasses):
- `Synapse` — directed edge between two memory IDs with strength ∈ [0, 1], citation type, decay rate, temporal bias
- `MemoryAugmented` — per-memory aggregate: incoming/outgoing strength, PageRank, hub score, importance score, replay buffer state
- `ReplayItem` — entry in the spaced-repetition replay queue with priority and due date
- `CitationType` enum — `RETRIEVAL`, `EXPLICIT`, `DECISION`, `CO_RETRIEVAL`, `MANUAL`, `TEMPORAL`

**Key subsystems:**

- **Strength dynamics** (`strength.py`): `on_citation()` boosts synapse strength based on citation type (MANUAL=1.0 > DECISION=0.5 > EXPLICIT=0.2 > CO_RETRIEVAL=0.1 > RETRIEVAL/TEMPORAL=0.05), modulated by STDP temporal bias. `apply_homeostatic_scaling()` caps total outbound strength per memory at 10.0 to prevent runaway.

- **Citation detection** (`synapse_tracker.py`): `CitationDetector` uses regex patterns for explicit citations and token-overlap Jaccard similarity (≥0.3 or ≥3 shared tokens) for semantic citations. `track_co_citations()` creates CO_RETRIEVAL synapses between all pairs of co-retrieved results with logarithmic strength boost.

- **Decay** (`decay.py`): `decay_all_synapses()` applies exponential decay weighted by source memory's importance score. Synapses below 0.2 get queued for replay; below 0.01 get deleted.

- **PageRank + HITS** (`pagerank.py`): `calculate_network_centrality()` runs weighted PageRank and HITS over the synapse graph, updates per-memory centrality and importance scores (importance = 0.5×PageRank + 0.3×authority + 0.2×hub).

- **Replay buffer** (`replay.py`): Spaced-repetition system for preventing catastrophic forgetting. `populate_queue()` identifies weakening synapses and isolated memories. `get_due_reviews()` returns prioritized items.

- **Visualization** (`visualization.py`): ASCII formatters for network graphs, strength reports, and metrics summaries.

**Data flow for `system.add_memory()`:**
1. Create/get `MemoryAugmented` record for the memory
2. If explicit `cited_ids` provided → create synapses + call `on_citation()` for each
3. If `context_memories` provided → `CitationDetector.detect_citations()` finds implicit citations via pattern matching and token overlap → create synapses + `on_citation()`
4. `apply_homeostatic_scaling()` normalizes outbound strengths

**Data flow for `system.on_search()`:**
1. Update access counts for all returned memory IDs
2. Create RETRIEVAL synapses between consecutive result pairs + `on_citation()`
3. `track_co_citations()` creates CO_RETRIEVAL synapses between all result pairs

**Environment:** `SYNAPTIC_DB_DIR` overrides default data directory. Default is `synaptic_memory/data/`.

### OpenMemory (`openmemory/`)

Self-hosted personal memory server with MCP support:

- **`api/`** — FastAPI app (`main.py`). SQLAlchemy models in `app/models.py` with SQLite. Alembic migrations. MCP server in `app/mcp_server.py` (FastMCP + SSE transport). OAuth 2.1 auth in `app/mcp_oauth.py`.
- **`ui/`** — Next.js 15 + React 19, Redux Toolkit, Tailwind, shadcn/ui. Uses pnpm.

**MCP server auth:** Token-based via `MCP_SERVER_AUTH_TOKEN` env var. OAuth 2.1 endpoints for Claude Cloud compatibility. `OAUTH_AUTO_APPROVE=true` skips the consent screen for single-user setups.

**Single-user setup:** `USER` env var in `api/.env` sets the default user identity. MCP tools fall back to this when no user_id is in the connection URL. `MEM0_DEFAULT_USER_ID` is passed through docker-compose to the memory client.

**Key models:** `User`, `App`, `Memory`, `Category`, `AccessControl`. Memories are auto-categorized via LLM on insert/update (SQLAlchemy event listeners in `app/models.py`).

**Docker Compose services:** `mem0_store` (Qdrant), `openmemory-mcp` (FastAPI on :8765), `openmemory-ui` (Next.js on :3012).

**Memory client init:** `app/utils/memory.py` — lazy singleton, auto-detects vector store from env vars, fixes Ollama URLs for Docker networking, parses `env:VAR_NAME` syntax in config values.

## Key Environment Variables

| Variable | Where | Purpose |
|---|---|---|
| `OPENAI_API_KEY` | `api/.env` | LLM + embeddings (default provider) |
| `USER` | `api/.env` + host env | Default user identity for OpenMemory |
| `MEM0_DEFAULT_USER_ID` | `api/.env` + docker-compose | SDK-level default user_id |
| `MCP_SERVER_AUTH_TOKEN` | `api/.env` | Auth token for MCP endpoints |
| `OAUTH_AUTO_APPROVE` | `api/.env` | Skip OAuth consent screen (`true` for single-user) |
| `MEM0_DIR` | host env | Override default `~/.mem0` storage directory |
| `SYNAPTIC_DB_DIR` | host env | Override default `synaptic_memory/data/` directory |

## Key Conventions

- Build system: **Hatch** (hatchling backend), but local dev uses `.venv` directly
- Linting/formatting: **ruff** (line-length 120, excludes `embedchain/` and `openmemory/`)
- Testing: **pytest** with pytest-mock and pytest-asyncio. Tests mirror source structure under `tests/`.
- Synaptic memory tests: all async, use `pytest_asyncio.fixture` with `tmp_path` for isolated SQLite DBs
- Python: 3.9+ required, type hints, Pydantic v2 for config models
- API responses use `{"results": [...]}` format (v1.1)
- Synaptic memory uses **aiosqlite** for async SQLite access, **dataclasses** (not Pydantic) for models
