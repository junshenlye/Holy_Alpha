# Architecture Decision Log

Append-only. Never modify existing entries. Add new entries at the bottom.

---

## 001 — Use Electron over Tauri
**Date:** 2026-02-15
**Context:** Need consistent cross-platform rendering for WebGPU and 3D visualization.
**Decision:** Electron 35+ (ships Chromium).
**Alternatives:** Tauri v2 (smaller bundle but WebKitGTK unstable on Linux, inconsistent WebGPU).
**Consequences:** Larger bundle (~80MB), higher memory, but predictable rendering and massive ecosystem.

## 002 — DuckDB-WASM as unified query engine
**Date:** 2026-02-15
**Context:** Need fast SQL queries on local CSV/Parquet/JSON files in-browser.
**Decision:** DuckDB-WASM as the query layer. All data sources load into DuckDB for querying.
**Alternatives:** SQLite-WASM (less analytical), raw Papa Parse (no SQL), Pandas only (Python-only).
**Consequences:** 6ms analytical queries. Adds ~10MB to bundle. Excellent Parquet support.

## 003 — Decoupled modules via event bus
**Date:** 2026-02-15
**Context:** Agents need to work on separate modules in parallel without conflicts.
**Decision:** All cross-module communication through a typed event bus. No direct imports.
**Alternatives:** Shared state store (coupling), direct imports (tight coupling).
**Consequences:** Slightly more boilerplate, but modules are independently testable and deployable.

## 004 — Claude Sonnet 4.5 as production LLM, Qwen 2.5 Coder 14B as local
**Date:** 2026-02-15
**Context:** Need LLM that excels at tool calling (MCP) and code generation.
**Decision:** Sonnet 4.5 for production (77% SWE-bench, #3 BFCL). Qwen 2.5 Coder 14B via Ollama for local (F1 0.971 tool calling, free, runs on 16GB Mac).
**Alternatives:** GPT-4.1 (solid but less capable for code), Claude Opus (too expensive for routine tasks).
**Consequences:** Requires API key for production. Local fallback is free but requires Ollama setup.

## 005 — Native Python child process over Pyodide-only
**Date:** 2026-02-15
**Context:** Pyodide (WASM) cannot connect to databases, limited to 4GB memory, missing packages.
**Decision:** Default to native Python child process spawned by Electron. Pyodide as optional fallback.
**Alternatives:** Pyodide-only (simpler but limited), remote Python server (latency).
**Consequences:** Requires Python bundling (~50MB) or user's system Python. Full package access.

## 006 — Yjs CRDT for concurrent user + LLM editing
**Date:** 2026-02-15
**Context:** Both user and LLM edit the same notebook. Need conflict-free concurrent editing.
**Decision:** Yjs (same CRDT library JupyterLab uses for real-time collaboration).
**Alternatives:** Operational Transform (needs central server), git branches per edit (too heavy for character-level).
**Consequences:** Character-level conflict resolution. Every edit tagged with author. Adds Yjs dependency.

## 007 — Conventional Commits with atomic commit strategy
**Date:** 2026-02-15
**Context:** Multiple agents work in parallel. Need clean git history and easy rollback.
**Decision:** Conventional Commits format. Atomic commits after each logical unit. Feature branches per milestone.
**Alternatives:** Squash merges (lose granularity), WIP commits (messy history).
**Consequences:** More commits, but each is self-contained and reviewable.
