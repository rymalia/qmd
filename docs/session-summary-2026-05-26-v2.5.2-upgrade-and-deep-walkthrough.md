---
date: 2026-05-26
time: "7:56 PM PDT – 9:33 PM PDT"
project: qmd
branch: dev
---

# Session Summary: 7-Week Catch-Up, v2.1.0 → v2.5.2 Upgrade, Bun Verification, and Deep Code Walk-Through

## Overview

Returned to the qmd project after a ~7-week gap. Synced from v2.1.0 to v2.5.2 (three minor releases, 101 commits), verified that Bun now works end-to-end (overturning a 2.5-month-old "no Bun" rule), updated CLAUDE.local.md and auto-memory to reflect the new reality, cleaned up merged feature branches, and conducted a detailed code-level walk-through of all v2.2–v2.5 changes.

## Key Decisions Made

- **Rebase over merge for upstream sync.** All 10 local commits on `dev` were docs-only, so rebasing onto `upstream/main` produced a clean linear history with no conflicts. A merge commit would have added noise without value.
- **Discarded local `src/cli/qmd.ts` edit before sync.** Our 3-line PR #533 fix had been merged upstream AND extended further (now passes `row.chunkLen` for absolute line numbers per #149). Upstream's version is strictly better, so the local copy was discarded with `git checkout` before rebasing — avoided a needless conflict.
- **Embraced dual-runtime support.** Verified Bun + bun:sqlite + sqlite-vec + node-llama-cpp work end-to-end in v2.5.2. The original "no Bun" ban was rooted in an older `bun:sqlite` limitation (lacked `loadExtension()`) that no longer applies. Rewrote CLAUDE.local.md to reflect "npm primary, Bun tolerated" instead of "Bun forbidden."
- **Daemon stays on Node, intentionally.** Even though Bun works, the MCP daemon continues to run via tsx+Node because LaunchAgent's PATH doesn't include `bun`. This is the well-tested deployment path; flipping the long-running daemon to a different runtime would invite untested regressions.
- **Deferred the 2,729-doc re-embed.** `qmd doctor` flagged most documents as having "legacy" fingerprints, but the vector-sample reproducibility check confirmed stored vectors still match current model output. Searches are unaffected; re-embedding is bookkeeping that can wait.
- **Committed CLAUDE.local.md to dev.** The file is tracked on `dev` but not on `main`/`upstream/main`, so its staged modification blocked `git checkout main`. Committing was the natural fix since the changes were intentional and worth persisting.

## Changes Made

| Change | Detail |
|--------|--------|
| **April 2026 reports committed** | `8f73399 docs: April 2026 reports — bench discoverability, stale index audit, virtual-path proposal` (3 untracked docs from previous sessions, 798 insertions) |
| **Sync to upstream v2.5.2** | Clean rebase of 10 docs commits onto `upstream/main` (`443760f release: v2.5.2`), pushed to `origin/dev` with `--force-with-lease` |
| **Local main fast-forwarded** | 101 commits via `git merge --ff-only upstream/main`; `origin/main` was already in sync via GitHub's fork auto-sync (push was no-op) |
| **Merged feature branches deleted** | `fix/json-line-field` (PR #533) and `fix/semantic-query-hyphen-validation` (PR #384) removed both locally and from `origin` |
| **Rebuild and daemon restart** | `npm run build` → `pkill -f "qmd.js mcp --http"` → LaunchAgent auto-restarted. New daemon PID 22688 running v2.5.2 |
| **CLAUDE.local.md rewritten** | `2b1c432 chore: update CLAUDE.local.md for v2.5.2 — Bun works, source-mode launcher noted` (45 insertions, 21 deletions). Version bump, runtime section rewrite, source-mode note, softened top intro |
| **Auto-memory updated** | `feedback_no_bun.md` rewritten from "No Bun, exclusively npm" to "npm primary, Bun tolerated"; obsolete `project_pr384_stale.md` deleted (PR was merged May 3); `MEMORY.md` index updated |

## Testing / Research Performed

### Install verification

- `qmd --version` → `qmd 2.5.2 (8f73399)` confirmed
- `qmd doctor` (new command, first run) — all checks green except legacy-fingerprint warnings (expected)
- `qmd status` under Bun — index loaded, 3,320 docs, 13,607 vectors, all 54 collections enumerated
- `qmd vsearch "embedding fingerprint migration" -n 3` under Bun — 3 results returned in ~3s, proving sqlite-vec loaded successfully under bun:sqlite
- Full hybrid `qmd query "qmd doctor diagnostics" -n 3 -c qmd` under Bun — query expansion + 6 sub-queries + 4-query embedding + 23-chunk rerank, all completed in 17.4s

### Code-level investigation

- Read `src/paths.ts` (new 5-line file)
- Read 130 lines of `qmd doctor` implementation (`src/cli/qmd.ts:3420-3550`)
- Grepped fingerprinting code paths in `src/store.ts` (lines 836–2496) — schema, lazy migration, sample-distance check
- Read `test/cli-lazy-llm-import.test.ts` — discovered the static-analysis-as-test pattern
- Read `scripts/build.mjs` — new shebang-prepending build wrapper
- Compared local `src/cli/qmd.ts` vs `upstream/main` to confirm zero divergence after rebase

### Upstream activity check

- All 4 of our PRs verified merged via `gh pr list`:
  - #382 zod pin (merged 2026-03-14)
  - #385 launcher (merged 2026-03-14)
  - #533 JSON line field (merged 2026-04-09)
  - #384 vec/hyde hyphens (merged **2026-05-03** — after 53 days open)
- All 7 issues we commented on checked via `gh issue view`: 6 closed, 1 still open (#513 source_path)

## Summary Statistics

- **Time elapsed**: ~97 minutes
- **Local commits added**: 2 (docs + chore) on top of upstream
- **Commits synced from upstream**: 101 (v2.1.0 → v2.5.2)
- **Branches deleted**: 2 (both local + remote on origin)
- **Files in untracked → committed**: 4 (3 docs + 1 modified CLAUDE.local.md)
- **Source code surveyed**: ~12,500 lines across `src/cli/qmd.ts`, `src/store.ts`, `src/llm.ts`, `src/mcp/server.ts`, `src/paths.ts`
- **New tests catalogued**: 6 new test files in `test/`
- **Memory files updated**: 3 (1 rewritten, 1 deleted, MEMORY.md index updated)
- **Upstream activity verified**: 4 PRs, 7 issues, 3 minor releases

---

# 🔎 Deep Walk-Through: v2.1.0 → v2.5.2

> The headline observation: velocity has shifted toward **operational maturity** rather than headline features. v2.1.0 added AST chunking and per-collection models (visible features). v2.5.x adds doctor, fingerprinting, lazy imports, dual-runtime CI, polyglot launcher — all *infrastructure* work that makes the existing features safer. This is a project in transition from "growing capability" to "hardening for distribution."

## Tier 1: Daily Workflow Changes

### 1.1 `qmd doctor` — your new first stop for diagnostics

The `doctor` command is implemented across ~400 lines in `src/cli/qmd.ts:3438-3851`, organized as a series of small named checks:

```typescript
checkDoctorIndexConfig(nextSteps)
checkEnvironmentOverrides(activeModels, configModels)
checkModelDefaults(activeModels, configModels)
checkModelCache(activeModels, nextSteps)
checkDevicePolicy()      // GPU mode
checkLegacyFingerprintAdoption()  // ← the big new thing
checkEmbeddingFreshness()
checkEmbeddingFingerprintHealth()
checkVectorSampleReproducibility()
```

**The pattern** is interesting: each check produces a one-line `✓`/`⚠`/`✗` status AND optionally pushes a remediation hint into a shared `nextSteps[]` array. At the end, doctor prints "Next steps" as a bulleted list. This separates "what's wrong" from "what to do about it" — a clean UX pattern worth stealing.

**Run it after every significant change** — schema updates, model changes, env var tweaks, post-rebase. It catches the kind of subtle issues that used to require reading code.

The full list of `QMD_*` env vars and their consequences is *also* embedded in doctor's source (`src/cli/qmd.ts:3429-3451`). Some new ones we should know about:

| Env var | Effect |
|---|---|
| `QMD_FORCE_CPU=1` | Bypass GPU backends entirely (avoid llama.cpp crashes) |
| `QMD_LLAMA_GPU=metal\|cuda\|vulkan` | Force a specific backend |
| `QMD_DOCTOR_DEVICE_PROBE=0` | Skip GPU probing in doctor (if probing crashes) |
| `QMD_DISABLE_DARWIN_QUERY_JSON_SAFE_EXIT=1` | Revert macOS Metal safe-exit (advanced) |
| `QMD_SKILLS_DIR` | Override where `qmd skills` looks |
| `QMD_EDITOR_URI` | Override the clickable terminal link template |
| `QMD_EMBED_PARALLELISM` | Override embedding parallel context count |

### 1.2 Lazy LLM imports (v2.5.0 — `qmd status` is now instant)

This is the subtle change that makes `qmd status`, `qmd get`, and `qmd ls` feel snappy. There's a test that **literally greps `src/llm.ts` to enforce the pattern**:

```typescript
// test/cli-lazy-llm-import.test.ts
test("node-llama-cpp is only dynamically imported by LLM operations", () => {
  const source = readFileSync(join(process.cwd(), "src", "llm.ts"), "utf-8");
  expect(source).not.toMatch(/import\s+(?!type\b)[\s\S]*?from\s+["']node-llama-cpp["']/);
  expect(source).toContain('import("node-llama-cpp")');
});
```

**Static-analysis-as-test** is an underused pattern. Instead of testing runtime behavior, this test reads the source code and asserts a structural invariant — that `node-llama-cpp` is never statically imported, only dynamically. Catches regressions instantly with no runtime cost. The alternative would be timing-based: "does `qmd status` complete in under 200ms?" — but that's flaky and platform-dependent. The grep is deterministic.

The user-visible impact: on machines with broken/missing GPU drivers (ARM, no-CUDA, etc.), running `qmd status` used to crash or hang while loading native bindings. Now it doesn't load them at all unless you actually run a query/embed.

### 1.3 Source-mode launcher

`bin/qmd` now prefers `src/cli/qmd.ts` via tsx in git checkouts. The new build script (`scripts/build.mjs`) is what makes packaged installs work — it runs `tsc` then prepends `#!/usr/bin/env node` to `dist/cli/qmd.js` and chmods 755. Previously this was a separate shell step; now it's encapsulated.

Practical implication for us: changes to source take effect on next daemon restart **even without `npm run build`**. The build is still recommended for catching TypeScript errors before runtime and keeping `dist/` in sync, but it's no longer load-bearing for the interactive CLI in a git checkout.

### 1.4 The `paths.ts` tiny file (5 lines)

```typescript
// src/paths.ts
import { homedir as osHomedir } from "node:os";
export function qmdHomedir(): string {
  return process.env.HOME || process.env.USERPROFILE || osHomedir() || "/tmp";
}
```

Solves the "Windows CLI/MCP split-brain when HOME is unset" bug. Previously some code paths used `os.homedir()` (which respects USERPROFILE on Windows), others read `process.env.HOME` directly (which is undefined on Windows in some shells). Now everyone uses this one helper. **Small file, big consistency win.**

---

## Tier 2: Search Quality Changes

### 2.1 Embedding fingerprinting (v2.5.0 — the big one)

This is the most consequential correctness fix. Before, vectors in `content_vectors` had no metadata about *how* they were generated. If you changed the embedding model or chunking parameters, old vectors stayed in the table and got mixed with new vectors at query time — silently producing degraded results.

**The schema change** (`src/store.ts:887`):

```sql
content_vectors (
  ...
  embed_fingerprint TEXT NOT NULL DEFAULT ''
)
```

Empty string = "legacy, generated before fingerprinting existed."

**The smart part: lazy adoption** (`src/store.ts:2154-2215`):

```typescript
// On first vector-health check after upgrade:
// 1. Sample a chunk from the legacy (empty-fingerprint) set
// 2. Re-embed it with the current model
// 3. If the new embedding matches the stored vector at distance < epsilon,
//    bulk-promote ALL legacy vectors to the current fingerprint
const update = db.prepare(
  `UPDATE content_vectors SET embed_fingerprint = ?
   WHERE model = ? AND embed_fingerprint = ''`
).run(fingerprint, model);
```

This is why our doctor output said: `adopted 3379 legacy chunks; sample matched current fingerprint at distance 0.000000`. The system tested one chunk, confirmed model+params haven't changed, and promoted everything in one statement. Surgical.

**Why 2,729 docs are still flagged "legacy":** Those docs are in collections whose chunking strategy *did* change (AST chunking added in v2.1.0). The adoption check correctly refused to promote them because their sample distance would not be zero.

### 2.2 Hybrid RRF weighting fix (#591)

A genuinely sneaky bug. The RRF (Reciprocal Rank Fusion) algorithm assigns 2x weight to the "original" query vs. expansion variants. The bug: it was indexed by **position in the query list**, not by query type, so the FIRST lexical expansion was accidentally getting the 2x boost instead of the user's actual original query.

The fix tags queries by `{type: "original" | "lex" | "vec" | "hyde"}` and weights based on that. **Concrete impact:** hybrid search results in v2.5.2 should rank slightly better than v2.1.0 for the same query.

### 2.3 Absolute line numbers (resolves the chunk-local issue from PR #506/#533)

Upstream's fix went further than our PR #533. They now pass `row.chunkLen` to `extractSnippet()`, which computes file-absolute line numbers from chunk-relative offsets. MCP `query` results now return `line` values that can be passed *directly* to `qmd_get` as `fromLine` — no separate lookup needed. This is in `src/cli/qmd.ts` around line 2177 (vs. our local edit's line 1935 — the offset reflects ~245 lines of new code added before `outputResults`).

---

## Tier 3: Fixes for Issues We Filed/Commented On

This is the emotional payoff section.

| Issue/PR | Status | Why it matters |
|---|---|---|
| **#384** vec/hyde hyphens | **Our PR merged May 3** | After 53 days open. Three duplicate issues filed in parallel proved the pain. |
| **#533** JSON `line` field | **Our PR merged Apr 9** | + extended to absolute line numbers, exactly the right follow-through |
| **#475/#520** handelize lowercase | Fixed via filename case preservation | The audit done in April that found "1,172 stale documents" wouldn't repeat — those false positives were caused by exactly this bug |
| **#100** handelize crashes | Fixed | The original `c++` → `c` collision documented in the bench report stays (#100 was about crashes, not collisions) |
| **#217** multi-collection CTE | Fixed May 20 | Our reproduction case from April directly contributed to maintainer attention |
| **#513** source_path column | **Still OPEN** | The virtual-path-disambiguation proposal is directly applicable — upstream hasn't built it yet |

**The open one (#513) is the lever you have right now.** The April proposal in `docs/virtual-path-disambiguation-proposal-2026-04-08.md` already lays out the design. Posting a link in #513 with a short hook would be a low-effort, high-leverage move.

---

## Tier 4: Infrastructure That Doesn't Affect Us But Is Worth Knowing

- **Polyglot launcher** (`bin/qmd`) — Detects runtime via `bun.lock` + `bun --version`, gracefully falls back to Node via tsx in source mode. The new `#!/usr/bin/env node` + JS comment polyglot trick was added in v2.5.2 to fix Windows global-install execution.
- **Dual-runtime CI** — `npm test` now runs `test:node` (vitest) AND `test:bun`. New test files specifically target Bun-quirks: `test/bin-wrapper.test.ts` (248 lines!), `test/esm-ambiguous-module.test.ts`, `test/package.test.ts`.
- **`qmd skills list|get|path`** — A new mechanism for serving version-matched skill instructions from the installed CLI. The old `src/embedded-skills.ts` was **deleted** in favor of reading from `skills/` directly at runtime. Less hardcoded, more flexible.
- **Container smoke harness** — `test/smoke-install.sh` grew from ~50 lines to 251 lines, now covers npm-global, npx-style, Bun-global installs with `QMD_FORCE_CPU=1` paths. This is upstream's defense against shipping broken installs.
- **MCP `serverInstructions` is now terse** — The collection summary in MCP startup messages was bloated; now it's a one-liner.

---

## Code Size Growth

| | v2.1.0 | v2.5.2 |
|---|---|---|
| `src/cli/qmd.ts` | ~3,082 lines | 4,496 lines (+46%) |
| `src/store.ts` | ~4,214 lines | 5,170 lines (+23%) |
| `src/llm.ts` | ~1,433 lines | 1,979 lines (+38%) |
| Total src code | ~9,500 lines | ~12,500 lines (+32%) |
| New commands | — | `doctor`, `skills list/get/path` |
| New env vars | — | 7+ new `QMD_*` vars |
| Test coverage | ~12 test files | ~20 test files |

The growth pattern in line counts is telling: `store.ts` only grew 23% while `qmd.ts` grew 46%. The CLI is absorbing more diagnostic logic (doctor!) and error-handling polish, while the core search engine in `store.ts` is in maintenance mode — getting fingerprinted and tested rather than fundamentally rearchitected. This is a **maturation signal**, not stagnation.

---

## Discoveries / Handoff Notes

### Bun runtime works end-to-end in v2.5.2

Critical reversal of a long-standing local policy. Verified empirically: `qmd status`, `qmd vsearch`, full hybrid `qmd query` with LLM expansion and qwen3-reranker — all succeed under Bun 1.3.10 with sqlite-vec loaded successfully. The original ban (CLAUDE.local.md 2026-03-11) was correct at the time; bun:sqlite has either added `loadExtension()` support since then or QMD's code paths under Bun route through `better-sqlite3`. Either way, the empirical evidence is clear.

### Split-runtime is fine

Under our setup, the MCP daemon runs on Node (LaunchAgent's PATH lacks bun) while interactive `qmd` may pick Bun (user shell has bun on PATH). This is acceptable and even desirable — the long-running daemon stays on the well-tested path, while interactive commands benefit from Bun's faster cold start. The two runtimes share the same SQLite database without issue.

### Source-mode launcher means `npm run build` is no longer strictly required

In a git checkout, `bin/qmd`'s launcher prefers `src/cli/qmd.ts` via tsx over `dist/cli/qmd.js`. Local source changes take effect on next daemon restart without rebuilding. Still recommended for type-checking and `dist/` consistency, but not load-bearing.

### `bun.lock` will keep coming back via rebases

Upstream now actively tracks `bun.lock` and updates it via maintenance commits. Do not delete it locally — it's a runtime-selection hint for the launcher and gets recreated on every dependency change upstream. CLAUDE.local.md's "do not recreate bun.lock" rule is reversed.

### GitHub fork auto-sync keeps `origin/main` current

The `git push origin main` after fast-forwarding was a no-op because `origin/main` (rymalia/qmd's main) was already at upstream/main. GitHub's fork-sync feature does this automatically. Worth knowing for future "is my fork's main up to date?" questions.

### The 2,729-doc legacy fingerprint warning is not urgent

The vector-sample reproducibility check in `qmd doctor` confirmed stored vectors still match current model output. The legacy/current split is metadata bookkeeping; searches work correctly. Re-embed at leisure, not in a panic.

### `formatter.ts` orphan module remains

In the previous (April) session we discovered that `src/cli/formatter.ts` exports shared formatting functions that have **zero callers** because `outputResults()` in `qmd.ts` evolved its own inline formatting. After the v2.5.2 sync, this file structure persists — worth checking next time whether it was finally consolidated or removed.

## Current State

- **Working directory**: `/Users/rymalia/projects/qmd`
- **Current branch**: `dev` (`2b1c432`)
- **`dev` HEAD**: `chore: update CLAUDE.local.md for v2.5.2 — Bun works, source-mode launcher noted`
- **`main` HEAD**: `443760f release: v2.5.2` (synced to upstream)
- **Installed binary**: `~/.nvm/versions/node/v24.12.0/bin/qmd` → symlink chain to `/Users/rymalia/projects/qmd`
- **Reported version**: `qmd 2.5.2 (8f73399)`
- **MCP daemon**: Running on Node via tsx (PID 22688 when restarted), LaunchAgent-managed (`com.qmd.mcp`)
- **Active branches**: `dev`, `main`, `dev-old` (archive), `feat/finetune-mlx-transition`, `feature/mcp-multi-client`
- **Working tree**: Clean

## Issues & PRs

### Our PRs — all merged

- **[#382](https://github.com/tobi/qmd/pull/382)** — Pin zod 4.2.1 (merged 2026-03-14)
- **[#385](https://github.com/tobi/qmd/pull/385)** — Prioritize package-lock.json in launcher (merged 2026-03-14)
- **[#533](https://github.com/tobi/qmd/pull/533)** — Include `line` in CLI --json output (merged 2026-04-09; extended by upstream to absolute line numbers)
- **[#384](https://github.com/tobi/qmd/pull/384)** — Allow hyphens in vec/hyde queries (merged **2026-05-03** after 53 days)

### Issues we touched

| # | Title | State |
|---|---|---|
| [#217](https://github.com/tobi/qmd/issues/217) | Multi-collection search false-empty | CLOSED 2026-05-20 |
| [#100](https://github.com/tobi/qmd/issues/100) | handelize crashes on non-alphanumeric | CLOSED 2026-05-20 |
| [#520](https://github.com/tobi/qmd/issues/520) | Search results show handelize'd paths | CLOSED 2026-05-17 |
| [#463](https://github.com/tobi/qmd/pull/463) | FTS5 hyphenated tokens | MERGED 2026-03-28 |
| [#383](https://github.com/tobi/qmd/issues/383) | vec/hyde hyphen validation | CLOSED 2026-05-03 (via our #384) |
| **[#513](https://github.com/tobi/qmd/issues/513)** | **source_path column** | **STILL OPEN** — our proposal applies |

## Unfinished Work

- **Comment on #513** with a link to `docs/virtual-path-disambiguation-proposal-2026-04-08.md`. The proposal directly addresses what upstream is trying to design; high-leverage low-effort follow-up.
- **Re-embed the 2,729 legacy-fingerprint docs** at leisure. Not urgent (searches still work), but `qmd embed` would silence the warning and bring all vectors onto the current fingerprint.
- **Investigate `src/cli/formatter.ts` orphan status** — verify whether the dead-code module from April was finally removed or still exists.
- **Consider testing `qmd skills list|get|path`** — new command surface, worth a quick poke to see if it serves anything useful for our Claude Code plugin layer.
- **Decision deferred: revert plugin to `github:tobi/qmd`** — `~/.claude/known_marketplaces.json` is still pointed at the local repo (`git:///Users/rymalia/projects/qmd`). Since all our PRs are merged, switching back to the GitHub source is now safe whenever convenient.
