# QMD Project Analysis: Evolution, Trends, and Documentation (2026-03-16)

This document provides a comprehensive analysis of the `tobi/qmd` repository, combining the detailed history from its `CHANGELOG.md` with insights from recent GitHub issues and pull requests, and a specific focus on user onboarding and documentation evolution.

### **Project Evolution & Major Themes**

The `qmd` project has evolved rapidly from a personal search tool into a robust, extensible, and community-driven hybrid search engine. The changelog reveals several key phases and architectural shifts:

1.  **Genesis (v0.1.0 - v0.5.0):** The project started as a single `qmd.ts` file, approximately 1800 lines of code, encompassing all functionality. It was built on a foundation of SQLite (FTS5 for keyword search and `sqlite-vec` for vector search). Initial versions focused on establishing the core hybrid search pipeline, using Ollama for LLM operations (embeddings, reranking, and expansion). A significant architectural change occurred with the move from SQLite-based configuration to a human-readable YAML file (`index.yml`), making the configuration more transparent and version-controllable.

2.  **In-Process LLMs and Agent Integration (v0.6.0 - v0.9.0):** A pivotal moment was the replacement of the Ollama server dependency with `node-llama-cpp`. This made `qmd` a zero-dependency tool by loading GGUF models directly in-process. This phase also saw the introduction of the MCP (Message Control Protocol) server, enabling AI agents (like Claude) to use `qmd` as a tool for knowledge retrieval. The search query language also became more sophisticated, with structured "query documents" (`lex:`, `vec:`, `hyde:`).

3.  **Stabilization, Performance, and SDK (v1.0.0 - v2.0.0):** This phase focused on maturing the project into a stable library. Key improvements include:
    *   **Cross-runtime Support:** Compatibility with both Node.js and Bun.
    *   **Performance:** Parallel GPU contexts for faster reranking and flash attention for reduced VRAM usage.
    *   **SDK First:** The architecture was refactored to establish a stable `QMDStore` SDK as the primary interface, with the CLI and MCP server becoming clean consumers of this API.

### **Deep Dive: Key Trends and Project Evolution**

This deeper analysis reveals a clear narrative of a project that has strategically evolved from a simple tool to a sophisticated, multi-faceted search platform.

#### **Trend 1: From Monolithic Script to Mature SDK**

The project's most significant architectural trend is its transformation from a single-file script into a stable, well-documented Software Development Kit (SDK).

*   **The Beginning (v0.1.0):** The project started as a single `qmd.ts` file, approximately 1800 lines of code, encompassing all functionality.
*   **Modularization (v0.4.0):** The first major refactor broke the monolith into focused modules like `store.ts`, `llm.ts`, and `mcp.ts`, and critically, introduced the project's first test suite. This was the first step toward creating a maintainable and extensible codebase.
*   **Library First (v1.1.6):** QMD became officially usable as a library. The `package.json` was updated to export `createStore`, allowing developers to integrate QMD's search capabilities directly into their applications without shelling out to the CLI.
*   **Stable SDK (v2.0.0):** This release marked the culmination of this trend. It declared a stable `QMDStore` API, cementing the SDK as the primary interface. The CLI and MCP server were refactored to be clean consumers of this public API. The **`README.md`** reflects this maturity with a comprehensive **"SDK / Library Usage"** section detailing everything from store creation to advanced search and indexing.

#### **Trend 2: Unrelenting Focus on Search Quality**

The core of QMD is its search pipeline, and its evolution shows a relentless pursuit of better relevance and power.

*   **Hybrid Foundation (v0.1.0):** The project began with its core idea: combining BM25 (keyword) and vector search with Reciprocal Rank Fusion (RRF).
*   **Sophisticated Query Language (v1.1.0):** The introduction of **"query documents"** and the `lex:`, `vec:`, and `hyde:` prefixes was a major leap. This gave users fine-grained control over the search process. The `docs/SYNTAX.md` file was created in this release to formally document this powerful new grammar.
*   **Disambiguation with `intent` (v1.1.5):** The `intent` parameter was introduced, allowing users to provide context to disambiguate vague queries (e.g., "performance" + `intent: "web page load times"`). This context steers every stage of the pipeline, from LLM expansion to reranking.
*   **Hardening and Edge Cases (Recent Activity):** The current work on **Issue #417** and **PR #418** to fix hyphen handling demonstrates a focus on hardening the engine. Making identifiers like `CVE-2024-1234` searchable is essential for a technical knowledge base and shows a commitment to real-world usability.

#### **Trend 3: Agent-First Design and Integration**

QMD was designed from early on to be a tool for AI agents, and this focus is clear in its development.

*   **Initial MCP Server (v0.4.0):** An MCP server was added to allow agents like Claude to interact with QMD programmatically, moving beyond simple CLI output parsing.
*   **Daemonization and Performance (v0.9.0):** The MCP server gained a `--http --daemon` mode, which was a critical improvement. It allowed the server to run in the background and keep models loaded in VRAM, slashing query latency for agents.
*   **Rich Instructions (v0.9.0):** The server began generating dynamic instructions for agents at startup, informing them about available collections and their contents, enabling smarter tool use. The latest `README.md` provides clear instructions for configuring the MCP server with Claude Desktop and Claude Code.

### **The Evolution of User Onboarding and Documentation**

A search of the project's history reveals that documentation has evolved in lockstep with features, focusing on clarity and practical examples for its key user groups: CLI users, agent developers, and SDK consumers.

While there are few issues or PRs *solely* dedicated to documentation, many feature and fix PRs include documentation updates as part of the change. This suggests a healthy practice of "documenting as you go."

*   **`README.md`: The Central Hub:** The README has grown from a basic quick start to a comprehensive portal. It now includes:
    *   A clear quick-start guide for immediate use.
    *   Detailed sections on agent integration (MCP Server) and library usage (SDK).
    *   An architectural diagram and explanation of the scoring and fusion strategy.
    *   A full reference for CLI commands, options, and environment variables.

*   **`docs/SYNTAX.md`: A Formal Grammar:** The creation of this file in **v1.1.0** was a significant step in user onboarding. As the query language grew more powerful with `lex:`, `vec:`, and `hyde:`, a formal reference became necessary. This document is crucial for users who want to move beyond simple keyword searches and unlock the full power of the hybrid engine.

*   **Changelog as a Narrative:** The `CHANGELOG.md` itself is a key onboarding asset. It's not just a list of commits; it's a well-written narrative that explains the *why* behind major changes, making it easy for users to understand the project's direction and the benefits of new versions.

*   **Evidence of Documentation Maintenance:**
    *   **PR #311 (in v1.1.2):** A community contribution fixed incorrect CLI commands in the README, showing that the documentation is actively maintained and valued by users.
    *   **PR #419 (Open):** The currently open PR to add the `--host` parameter includes corresponding documentation updates, continuing the practice of bundling code and doc changes together.

In summary, the project's documentation has matured alongside the codebase. It effectively serves both new users looking for a quick start and advanced users or developers needing deep technical details, with the `README.md` as the entry point and the `CHANGELOG.md` and `docs/SYNTAX.md` as essential references for deeper understanding.