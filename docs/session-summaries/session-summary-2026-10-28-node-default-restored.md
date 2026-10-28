---
session_id: 225822b6-5140-414a-9f92-98b2b7dd1dc4
date: 2026-10-28
time: "2:02 AM PDT – 2:25 AM PDT"
project: qmd
branch: dev
---

# Session Summary: qmd back on PATH (nvm default restored to Node 24.22.1)

> **Metadata note:** this session ran from the `~/projects` workspace, not inside this repo.
> This file tells qmd about a machine-level config change; there were no code changes to qmd.
> The full session record is
> `~/projects/docs/session-summaries/session-summary-2026-10-28-node-default-back-to-24.md`.

## Overview

`qmd` stopped resolving in shells because the nvm default was switched to Node 26.12.1 on
2026-10-27 (in a birdclaw session). qmd is installed only under v24.22.1. The default was set
back to **24.22.1**, and `qmd` now works again in new shells.

## Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| **Restore the default rather than install qmd under v26** | Every global CLI lives under v24. The LaunchAgent is pinned to v24.22.1 already. A second qmd install would let the shell and the daemon drift apart, with native addons built for separate copies |
| **Leave the plist as is** | It points at `v24.22.1/bin/qmd` in both `ProgramArguments` and `PATH`, which is the version that's now the default again |

## Changes Made

| Change | Detail |
|--------|--------|
| **nvm default (outside the repo)** | `~/.nvm/alias/default`: `26.12.1` → `24.22.1` |
| **This summary** | No other files in this repo were touched. The pre-existing modification to `docs/node-101-package-managers.md` is not from this session and was not committed |

## Testing / Research Performed

- **Clean-env login shell** (`env -i … zsh -lic`):
  - `which qmd` → `~/.nvm/versions/node/v24.22.1/bin/qmd`; `qmd --version` → `qmd 2.8.3 (facd35e)`.
  - `qmd doctor` reported all ✓: model cache valid, Metal GPU on Apple M3, embeddings fresh, 6,270 docs on fingerprint `c37385`, sampled vectors reproduce.
- **The daemon stayed up throughout.** The plist pins v24.22.1, and the `mcp__qmd__*` tools stayed available during the v26 period.
- **Codex gpt-6-luna** found that the qmd *plugin's* MCP entry runs a bare `"command": "qmd"` from PATH. That's why `plugin:qmd:qmd` failed with "Executable not found in $PATH" while the HTTP daemon kept working.
- **Black October re-check on both Nodes:** Brotli `fix=3`, eMudhra 14/14 rejected. v24.22.1 is still a patched version.

## Summary Statistics

- qmd files changed: 0 (only this summary)
- Config settings changed: 1 (nvm default alias, machine level)

## Discoveries / Handoff Notes

- **Two ways qmd gets launched, and they fail differently:**
  - The LaunchAgent uses an absolute path, so it survives a change of nvm default but breaks when that exact version is uninstalled.
  - The plugin MCP entry uses a bare `qmd` from PATH, so it breaks whenever the default Node lacks a qmd install.
- **The pasted qmd-agent explanation had the date wrong.** The default changed on Oct 27 at 15:19, not Oct 23.
- **Its ABI warning was right:** don't symlink the v24 `qmd` onto a v26 PATH. Its `#!/usr/bin/env node` would load v24-built `better-sqlite3`/`node-llama-cpp` under v26.

## Unfinished Work

- Restart Claude Code so `plugin:qmd:qmd` reconnects. It was still failing to connect in this session.
- Still open: the plist pins `v24.22.1`, so the next Node upgrade must repoint it (or switch it to a stable symlink/wrapper).
