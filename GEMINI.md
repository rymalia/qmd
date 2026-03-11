# GEMINI.md

## Project Overview
**QMD (Query Markup Documents)** is an on-device hybrid search engine designed for markdown knowledge bases, notes, and documentation. It combines BM25 full-text search (SQLite FTS5), vector semantic search (sqlite-vec), and local LLM re-ranking (node-llama-cpp) to provide high-quality retrieval.

The project includes a comprehensive **Fine-Tuning Suite** (`finetune/`) dedicated to training small language models (primarily Qwen3-1.7B and LiquidAI LFM2) to expand raw user queries into structured "Query Documents" containing lexical, vector, and HyDE signals.

### Main Technologies
- **Core Engine:** TypeScript, Bun/Node.js, SQLite (FTS5 + `sqlite-vec`), `node-llama-cpp`.
- **Fine-Tuning:** Python (`uv`), `transformers`, `peft`, `trl` (SFT & GRPO), `dspy-ai`.
- **Models:** EmbeddingGemma (embeddings), Qwen3-Reranker (re-ranking), fine-tuned Qwen3 (query expansion).

## Building and Running

### Core QMD Tool
- **Install Dependencies:** `bun install`
- **Build from Source:** `bun run build`
- **Run Locally:** `./qmd <command>` or `bun run qmd -- <command>`
- **Index/Update:** `qmd collection add <path>` followed by `qmd embed` and `qmd update`.
- **Search:** `qmd query "your search query"` (Hybrid) or `qmd search "keywords"` (BM25).
- **MCP Server:** `qmd mcp` (stdio) or `qmd mcp --http --daemon` (warm server).
- **REST API:** The HTTP server also exposes `POST /query` (and alias `/search`) for direct search without MCP. See README.md for request format.
- **Restart (daemon mode):** `qmd mcp stop && qmd mcp --http --daemon`
- **Restart (LaunchAgent):** `launchctl bootout gui/$(id -u)/com.qmd.mcp && launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist`
- **Kill orphaned process:** `lsof -ti :8181 | xargs kill` (use when `qmd mcp stop` says "no PID file" but port is occupied)
- **Health check:** `curl http://localhost:8181/health`
- **Note:** `qmd mcp stop` only works for `--daemon` mode (PID file). LaunchAgent processes must use `launchctl bootout`.
- **Session errors**: If you get `404 "Session not found"` or `"Session expired"`, the server restarted and your client has a stale session ID. **Reconnect the client, don't restart the server** — the server is healthy. Restart your Gemini CLI session to re-initialize. Verify the server is up with `curl http://localhost:8181/health`.
- **If the server is actually down**: Restart it with `qmd mcp stop && qmd mcp --http --daemon` (daemon mode) or `launchctl bootout gui/$(id -u)/com.qmd.mcp && launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.qmd.mcp.plist` (LaunchAgent).
- **CRITICAL:** Do NOT run vector-related commands via `bun src/qmd.ts`. `bun` currently lacks support for dynamic extension loading for `sqlite-vec` on macOS. Always use `./qmd` or `npx tsx src/qmd.ts`.
- **Deploying source changes:** After modifying source files, rebuild and relink for the LaunchAgent to pick up changes: `bun run build && bun link`, then restart the LaunchAgent. The global `qmd` binary (`~/.bun/bin/qmd`) is a symlink managed by `bun link` — without relinking, the LaunchAgent runs stale code.
- **Multi-client:** The HTTP server supports multiple simultaneous MCP clients (Claude Code + Gemini CLI) via per-session architecture.

### Fine-Tuning Suite (`finetune/`)
- **Data Prep:** `uv run dataset/prepare_data.py` (combines `.jsonl` files in `data/`).
- **SFT Training:** `uv run train.py sft --config configs/sft.yaml`
- **GRPO Training:** `uv run train.py grpo --config configs/grpo.yaml`
- **Evaluation:** `uv run eval.py --model ./outputs/grpo`
- **GGUF Conversion:** `uv run convert_gguf.py --size 1.7B`

## Development Conventions

### Engineering Standards
- **Runtime Compatibility:** Code must support both **Bun** and **Node.js (>=22)**.
- **Search Pipeline:** Adhere to the hybrid RRF fusion logic (Original query ×2, Top-rank bonus, Position-aware blending).
- **Query Documents:** Use the `lex:`, `vec:`, `hyde:`, and `expand:` prefix convention for structured queries.
- **Testing:** New features or fixes MUST include tests in the `test/` directory. Run via `bun test`.
- **Releases:** Use the `/release <version>` skill. Never commit `[Unreleased]` changes manually; follow the `CHANGELOG.md` standards.

### Data & Model Rules
- **Named Entity Preservation:** Every `lex:` expansion line MUST preserve entities from the original query to prevent generic drift.
- **Model Deployment:** Never push a fine-tuned model to HuggingFace without running `eval.py` and verifying an improvement in average score.

## Session Continuity & Core Mandates

### Session Continuity (READ THIS FIRST)
**At the start of every new session**, check for recent session summary documents:
```bash
ls -t docs/session-summary-*.md | head -1
```
If a session summary exists, **read it immediately**. These contain unfinished work, current state, and context to prevent repeating investigations. **Confirm you have reviewed the summary in your opening greeting.**

### GIT USAGE RESTRICTIONS (IMPORTANT!)
- **NEVER RUN A GIT COMMIT COMMAND.**
- The user performs commits personally. Suggest the exact `git` commands and wait for direction.
- Always propose a draft commit message focused on "why" rather than "what".

### Session Summaries
At the end of each session, generate a comprehensive summary in `docs/session-summary-YYYY-MM-DD-{DESCRIPTOR}.md` covering:
- Key decisions and rationale.
- Files modified and issues fixed.
- Unfinished work and specific next steps.
- Context that prevents repeating already-completed investigations.
