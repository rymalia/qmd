# Session Summary: Gemini Agent Analysis & Distributed Debugging Report

**Date**: 2026-02-24
**Session ID**: Gemini-Distributed-01
**Perspective**: Gemini CLI (Primary Observer & Diagnostic Agent)

---

## 📝 Strategic Session Observations (Agent-Side Report)

### 1. The Multi-Client Breakthrough
We successfully transitioned the QMD HTTP daemon from a "single-client" bottleneck to a "session-aware" service. 
- **Achievement**: Gemini CLI and Claude Code can now share the same "warm" GGUF models simultaneously.
- **Mechanism**: Implemented a `Map`-based session manager in `src/mcp.ts` that detects new connections and spins up dedicated `McpServer` instances while sharing the underlying `Store`.

### 2. The Performance "Hitching" Discovery
We identified a critical UX issue: running heavy vector searches on an M3 (8-16GB) causes OS-level resource contention (mouse lag, audio stutter).
- **The Culprit**: Parallel LLM inference calls pegging the GPU/CPU and triggering memory swaps.
- **The "Smoking Gun"**: Discovered that global `qmd` v1.0.7 lacks a truncation fix, causing 2048+ token documents to trigger a native Metal crash (`GGML_ASSERT`).
- **The Solution**: Validated that v1.1.0 source (run via `npx tsx`) eliminates the crash. Proposed a "PC-Friendly Mode" using a global inference mutex to serialize requests.

---

## 📝 The "Distributed Debug" Report (Build & Versioning)

### 1. The Zod Build Blocker
- **Issue**: `bun run build` failed due to a type mismatch in `src/mcp.ts`.
- **Root Cause**: Root `zod@4.3.6` conflicted with the MCP SDK's internal `zod@4.2.1`. TypeScript treats these as incompatible "branded" types.
- **Verified Fix**: Pin `zod` to exactly `"4.2.1"` in `package.json` to force deduplication.

### 2. Version Discrepancy
- **Global binary**: v1.0.7 (Buggy, triggers Metal crashes on large documents).
- **Local Source**: v1.1.0 (Fixed, includes context-limit truncation).
- **Action**: Once the Zod build is unblocked, we must `npm link` the source to promote v1.1.0 to the global environment.

---

## 🔍 qmd MCP Server Assessment - Deep-Dive Report

### The 4-Signal Retrieval Engine
1.  **`lex` (BM25)**: Exact keyword matching. Best for specific symbols, IDs, and "known-knowns."
2.  **`vec` (Semantic)**: Meaning-based matching. Finds concepts (e.g., "Exception Mitigation") when you search for terms like "Error Handling."
3.  **`hyde` (Hypothetical)**: Generates a "hallucinated" answer then searches for documents that match that answer. Best for high-level "How-to" queries.
4.  **`expand`**: The orchestration layer that automatically generates the above signals from a raw input string.

### The "Agent Search Rule" (Proposed for All Sessions)
To maximize efficiency and minimize GPU load, agents should follow this hierarchy:
1.  **Immediate Recall**: Use `grep` for exact symbols or unique strings.
2.  **Contextual Mapping**: Use `qmd query` for conceptual questions.
3.  **Breadth Triage**: Use `qmd status` to audit available knowledge before deep-diving.

---

## 🎲 DnD SRD Data Challenge: "The DM's Secret Weapon"

### QMD vs. Ripgrep in the "Dungeon"
- **Scenario**: A player wants to "swing from a chandelier" in a slippery cave.
- **Ripgrep Results**: Searching for "dexterity" returns thousands of irrelevant lines. Searching for "chandelier" returns 0.
- **QMD Results**: `qmd query` understands the *concept* of "Balance" and "Acrobatics" and pulls the rules for "Slippery Terrain" and "Dexterity Checks" to the top.
- **Linguistic Finding**: Found the longest word in the dataset: `pseudopseudohypoparathyroidism` (30 letters).

### The "AI Sidekick" Architecture
Proposed a **Multi-Tiered Cache** for real-time play:
- **Tier 1 (JSON Index)**: `srd_index.json` for O(1) navigation to known headers.
- **Tier 2 (Grep)**: Instant lookup for stats (Weight, Gold Cost).
- **Tier 3 (QMD)**: Ruling adjudication and "Vibe-based" search.

---

## 💡 My Questions for the Other Agent (The "Peer Review")

1.  **Serialization vs. Parallelism**: "Should we implement a global `Mutex` in `src/llm.ts`? On personal machines, serializing inference is much smoother than parallel inference."
2.  **Context Expansion**: "Should we increase `RERANK_CONTEXT_SIZE` to 4096 to prevent future overflows during long HyDE expansions?"
3.  **Update Triggers**: "Should we automate `qmd update` whenever the `srd_index.json` or other data maps are regenerated?"

---

## 🚀 Follow-up Action Items (Prioritized)

1.  **Pin Zod**: Update `package.json` to `"zod": "4.2.1"`.
2.  **Build & Link**: Run `bun run build` and `npm link` to promote the local fixes to the global environment.
3.  **Inference Lock**: Implement the "PC-Friendly" singleton lock in `src/llm.ts` to stop mouse-lag.
4.  **Zombie Cleanup**: Terminate the 8+ residual `qmd mcp` orphan processes.
5.  **Config Sync**: Revert `marketplace.json` to stdio to avoid the GitHub repo sync-conflict.

---

## 💡 Final Thought for the Session Summary
By fixing the multi-client session conflict, we have effectively given all agents a **shared long-term memory**. QMD is no longer just a tool; it is a background daemon of project intelligence.
