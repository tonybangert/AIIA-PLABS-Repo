# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

AIIA — AI Information Architecture. A persistent-memory runtime that runs alongside Claude Code via MCP, plus a web dashboard. Two codebases live here:

- `local_brain/` — Python 3.10+ package (FastAPI services, MCP server, ChromaDB, Ollama client)
- `dashboard/` — React 19 + TypeScript + Vite frontend that talks to the Python services

The Python package is the source of truth. The React dashboard is a separate npm project under `dashboard/`.

## Common commands

### Python (`local_brain/`)

```bash
# Install (editable + dev tools)
pip install -e ".[dev]"

# Lint / format (ruff is authoritative — pyproject.toml configures it; ignore CONTRIBUTING.md's black/isort references, they're stale)
ruff check local_brain/
ruff format --check local_brain/
ruff check local_brain/ --fix       # autofix
ruff format local_brain/             # apply formatting

# Tests — note these are integration tests that require Ollama + Brain API + Command Center running.
# CI only runs --collect-only; running them locally requires `brain start` first.
pytest local_brain/tests/
pytest local_brain/tests/test_autonomy.py            # single file
pytest local_brain/tests/test_autonomy.py::test_name # single test
pytest --collect-only local_brain/tests/             # what CI does — verifies imports only

# Run services directly (without the brain CLI)
python -m local_brain.local_api                # Brain API on :8100
python -m local_brain.command_center.server    # Command Center on :8200
python -m local_brain.mcp_server               # MCP server (stdio, for Claude Code)
python -m local_brain.eq_brain.bootstrap       # Re-index knowledge into ChromaDB
```

### Dashboard (`dashboard/`)

```bash
cd dashboard
npm install
npm run dev      # Vite dev server
npm run build    # tsc -b && vite build
npm run lint     # eslint
```

### Security & local automation

```bash
scripts/security_scan.sh           # Full 7-scanner suite → ./security-reports/<date>/
scripts/security_scan.sh --quick   # Secrets + deps only
```

## CI gates (.github/workflows/ci.yml)

Two jobs run on push to `main` and on every PR:

1. **lint-and-test** — `ruff check`, `ruff format --check`, `pytest --collect-only`. All three are `continue-on-error: true` today (existing backlog), so CI won't fail on findings — but new code should be clean. The hard gate is `from local_brain.__version__ import __version__` actually importing.

2. **sanitization-guard** — Hard fail if any banned string reappears anywhere in the repo (case-insensitive). Banned list lives in `.github/workflows/ci.yml`. Excluded files: `CHANGELOG.md`, `PULL_REQUEST_TEMPLATE.md`, `ci.yml`, `package-lock.json`. **Before adding any text that mentions former proprietary product/client names, check ci.yml's banned regex** — adding such a string anywhere else will break the build.

## Architecture

This is a multi-process system, not a single app. The pieces talk to each other over HTTP and through shared JSON files on disk.

### Three long-running services

| Service | Port | Entry point | Role |
|---------|------|-------------|------|
| Ollama | 11434 | external (brew/docker) | Local LLM inference (llama3.1:8b, nomic-embed-text, deepseek-r1:14b) |
| Brain API | 8100 | `local_brain/local_api.py` | Core AIIA query/memory/knowledge endpoints |
| Command Center | 8200 | `local_brain/command_center/server.py` | Dashboard UI, task scheduler, action queue, execution engine, WebSocket |

A fourth process is the **MCP server** (`local_brain/mcp_server.py`) which Claude Code spawns over stdio per `.mcp.json`. The MCP server is a thin wrapper that calls into the Brain API and Command Center over HTTP.

### The two halves of `local_brain/`

- **`eq_brain/`** — Stateful intelligence. `brain.py` is the AIIA class; `memory.py` reads/writes 9 typed JSON files; `knowledge_store.py` wraps ChromaDB; `smart_conductor.py` (one level up) routes queries by complexity to local/single-call/agentic-loop paths; `recursive_engine.py` is the multi-step RLM loop. State lives in `EQ_BRAIN_DATA_DIR` (default `~/.aiia/eq_data`).

- **`command_center/`** — Operations. `server.py` is the dashboard FastAPI app. `aiia_tasks.py` is the scheduled-task runner (11 tasks on cron-like intervals). `action_queue.py` manages the pending→approved→executing→completed lifecycle for fixes the system wants to apply. `monitor_data.json`, `task_data.json`, `action_data.json` are the persistence layer.

### Execution engine (`local_brain/execution/`)

When the system wants to *do* something (run lint, commit, apply a fix), it goes through this pipeline:

1. **`story_executor.py`** decomposes a story into 2–8 actions via Ollama.
2. Each action lands in the action queue (pending).
3. **`safety.py`** assigns a tier: AUTO (run now), SUPERVISED (30s delay + notify), GATED (requires manual approval). `security_fix` and `review` are always GATED. Forbidden files (`.env*`, `*.pem`, `*.key`, `render.yaml`, anything under `*/migration/*`) cannot be touched.
4. Approved actions execute via one of three strategies in **`strategies.py`**: `DirectFixStrategy` (ruff), `ClaudeCodeStrategy` (spawns Claude CLI on an `aiia/*` branch), `CommitStrategy` (git add+commit).
5. **`verification.py`** post-checks; **`execution_log.py`** records history.

Concurrency is hard-capped at 1. Max 20 files per action. Don't add features that loosen these without explicit discussion.

### MCP integration

The 15 `aiia_*` tools exposed to Claude Code (see README.md for the full list) follow a session protocol: `aiia_session_start` loads context, `aiia_remember` / `aiia_log_story` capture during work, `aiia_session_end` records summary and auto-extracts stories from `next_steps`/`blockers` fields. Stories get dedup'd against the existing roadmap via SequenceMatcher at 85% similarity.

### Versioning

`local_brain/__version__.py` is the **single source of truth** for version. Brain API, Command Center, and dashboard all read from it. CI explicitly verifies the import works and the value is non-empty. Don't add a separate version constant elsewhere.

## Conventions

- **Ruff config:** line length 100, target py310, rules `E,F,I,UP,B,SIM`. Tests get `E501,F401` ignored. (CONTRIBUTING.md mentions black/88-char — that's outdated; pyproject.toml wins.)
- **JSON over a database.** Memory, stories, actions, monitor data are all JSON files. This is intentional (portable, git-diffable, zero-config). Don't introduce SQLite/Postgres without discussing.
- **Local-first, graceful degradation.** The system must work with only Ollama running. `ANTHROPIC_API_KEY` and `GOOGLE_API_KEY` are optional fallbacks.
- **No new exec actions default to AUTO.** New action types start at SUPERVISED or GATED.
- **Branch naming for execution-engine work:** `aiia/*` (the engine creates these). Human work follows `feat/`, `fix/`, `chore/`, etc.

## Important paths

- `.mcp.json` — Template for Claude Code MCP wiring (paths are placeholders).
- `.security-baseline.json` — Accepted findings; security scan only fails on *new* findings. Workflow doc: `docs/security.md`.
- `docs/long-context.md` — How to enable 64K context via Ollama flash attention + 4-bit KV cache.
- `~/.aiia/eq_data/` — Runtime data dir (memory, ChromaDB, roadmap, reports). Configurable via `EQ_BRAIN_DATA_DIR`.
