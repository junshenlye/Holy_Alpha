# Holy Alpha - System Design

## Architecture Overview

Holy Alpha is a standalone Electron desktop application with four decoupled modules.
Each module is independently buildable, testable, and shippable.

```
┌──────────────────────────────────────────────────────────┐
│                    ELECTRON SHELL                         │
│                                                          │
│  ┌────────────┐ ┌────────────┐ ┌────────┐ ┌──────────┐ │
│  │  NOTEBOOK   │ │    DATA    │ │  VIZ   │ │   CHAT   │ │
│  │  MODULE     │ │   MODULE   │ │ MODULE │ │  MODULE  │ │
│  └──────┬─────┘ └──────┬─────┘ └───┬────┘ └────┬─────┘ │
│         │              │           │            │        │
│         └──────────────┴───────────┴────────────┘        │
│                        │                                  │
│              ┌─────────▼──────────┐                       │
│              │    EVENT BUS       │                       │
│              │  (loose coupling)  │                       │
│              └─────────┬──────────┘                       │
│                        │                                  │
│              ┌─────────▼──────────┐                       │
│              │   RUNTIME ENGINE   │  (separate process)   │
│              └─────────┬──────────┘                       │
│                        │                                  │
│              ┌─────────▼──────────┐                       │
│              │   MCP LAYER        │  (LLM integration)    │
│              └────────────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

### Why Decoupled Modules

Each module communicates through an **event bus** (not direct imports).
This means:
- Notebook module works without Viz module
- Data module works without Chat module
- Any module can be developed and tested in isolation
- Agents can work on separate modules in parallel without conflicts
- A bug in Viz doesn't break Notebook

**Event examples:**
```typescript
// Data module emits when data is loaded
bus.emit("data:loaded", { tableId: "sales", schema: {...}, rowCount: 1000 })

// Notebook module listens to generate a preview cell
bus.on("data:loaded", (payload) => { /* create preview cell */ })

// Viz module listens to offer chart suggestions
bus.on("data:loaded", (payload) => { /* suggest chart types */ })

// Each listener is independent. Remove one, others still work.
```

---

## Module Breakdown

### Module 1: Notebook
The core editing surface. Jupyter-style cells (code, markdown, formula).
- Monaco editor for code cells
- KaTeX rendering for formula cells
- Yjs CRDT for concurrent editing (user + LLM)
- Cell dependency DAG for reactive execution
- Cells stored as structured data (not .ipynb JSON blob)

### Module 2: Data
Data loading, browsing, and querying.
- CSV file import (drag-and-drop, file picker)
- DuckDB-WASM as the query engine (SQL on local files)
- Schema browser (columns, types, stats, preview)
- Data verification panel (see your data before analysis)

### Module 3: Viz
2D and 3D visualization rendering.
- Plotly.js for 2D charts (declarative JSON → chart)
- Three.js for 3D scenes (WebGPU primary, WebGL fallback)
- Receives data from event bus, renders in panel
- Dimension ladder: any 2D chart can be promoted to 3D

### Module 4: Chat
Conversation interface with LLM.
- Chat panel alongside notebook
- MCP client connecting to LLM providers
- LLM reads notebook context, proposes cells
- User reviews proposals before execution

---

## LLM Strategy

### Model Selection

| Role | Model | Why | Cost |
|------|-------|-----|------|
| **Development/Testing** | Claude Haiku 4.5 | Fast, cheap, supports tool use + MCP | $1/$5 per M tokens |
| **Production** | Claude Sonnet 4.5 | Best code gen (77% SWE-bench), #3 BFCL for function calling, native MCP | $3/$15 per M tokens |
| **Local/Offline** | Qwen 2.5 Coder 14B | F1 0.971 tool calling, runs on 16GB Mac, free | $0 (local) |
| **Budget fallback** | GPT-4.1 | Solid tool calling, cheapest commercial | $2/$8 per M tokens |

### Why Claude Sonnet 4.5 for Production
- Ranks #3 on Berkeley Function Calling Leaderboard (70.29%)
- 77.2% on SWE-bench (code generation benchmark)
- Native MCP support (designed for tool-use workflows)
- 200K context window (1M beta) — handles large notebook state
- Prompt caching: 90% discount on repeated context ($0.30/M for cached input)
- Streaming support for real-time code generation

### Why Qwen 2.5 Coder 14B for Local
- F1 score 0.971 on function calling (nearly matches GPT-4)
- Runs on 16GB M-series Mac (Q4_K_M quantization)
- 128K context window
- Native function calling (no special prompting)
- Via Ollama: `ollama pull qwen2.5-coder:14b-instruct-q4_K_M`

### Context Management
- Keep active context under 50-80K tokens for reliability
- Send minimal initial context: notebook metadata + cell index (not full code)
- Fetch full cell content on-demand via MCP tools
- Use prompt caching for repeated schema context (90% cost reduction)
- Sliding window: last 5-10 exchanges verbatim, older → summarized

---

## MCP Architecture

### Holy Alpha as MCP Server (tools the LLM can call)

```typescript
// Core tools exposed to the LLM:

"create_cell"     — Create a new code/markdown/formula cell
"edit_cell"       — Modify an existing cell's content
"delete_cell"     — Remove a cell
"run_cell"        — Execute a cell and return output
"read_cell"       — Read a cell's source and output
"list_cells"      — List all cells with type and preview
"query_data"      — Run SQL on loaded data (DuckDB)
"get_schema"      — Get schema of loaded datasets
"create_viz"      — Generate a visualization spec (Plotly JSON)
"get_variables"   — Inspect runtime variable values
```

Each tool has:
- JSON Schema input validation (via Zod)
- Detailed description telling the LLM WHEN to use it
- Error responses with `isError: true` so LLM can recover
- Flat schemas (no deep nesting) to minimize tokens

### Holy Alpha as MCP Client (connects to LLMs)

```
Providers supported:
├── Claude API (Anthropic) — primary, native MCP
├── OpenAI API — via function calling translation
├── Ollama (local) — via stdio transport
└── LM Studio (local) — via OpenAI-compatible API
```

Transport: **stdio** (standard for desktop apps, lowest latency, simplest setup).

### MCP TypeScript SDK

Using `@modelcontextprotocol/sdk` v1.x (stable, production-ready).
v2.x expected Q1-Q2 2026 — only import paths change, concepts identical.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "holy-alpha", version: "0.1.0" });

server.registerTool("query_data", {
  description: "Run SQL query on loaded CSV data. Use when user asks questions about their data.",
  inputSchema: {
    sql: z.string().describe("SQL query to execute against DuckDB"),
    limit: z.number().optional().default(100).describe("Max rows to return")
  }
}, async ({ sql, limit }) => {
  try {
    const result = await duckdb.query(sql, limit);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  } catch (err) {
    return { content: [{ type: "text", text: `SQL error: ${err.message}` }], isError: true };
  }
});
```

---

## Milestones

### Milestone 0: Project Scaffolding
**What:** Empty Electron + React + TypeScript app that launches.
**Success criteria:**
- `npm run dev` opens an Electron window with a React page
- Hot reload works
- Event bus module exists with emit/on/off
- Project structure follows monorepo layout (packages/)
- CLAUDE.md initialized with project context
- CI runs: lint + type-check pass

**Depends on:** Nothing. This is the foundation.

### Milestone 1: Notebook Module (Core)
**What:** A working cell-based editor. Code and markdown cells.
**Success criteria:**
- User can create, edit, delete, reorder cells
- Code cells use Monaco editor with Python syntax highlighting
- Markdown cells render with a markdown renderer
- Cells persist to disk (save/load .holy files)
- Cell structure: `{ id, type, source, outputs, metadata }`
- Unit tests for cell CRUD operations
- No dependency on any other module

**Depends on:** Milestone 0

### Milestone 2: Runtime Engine
**What:** Python execution of code cells with dependency tracking.
**Success criteria:**
- Python child process spawns from Electron main
- IPC protocol: send code → receive output (stdout, stderr, result)
- Cell dependency DAG: static analysis of variable defs/refs
- Reactive execution: edit cell A → re-runs dependents of A
- Variable inspector: query current value of any variable
- Handles execution errors gracefully (shows traceback in cell)
- Runtime state is reproducible from cell code (no hidden state)
- Unit tests for DAG construction and execution order

**Depends on:** Milestone 1 (needs cells to execute)

### Milestone 3: Data Module (CSV Only)
**What:** Load CSV files and query them with SQL.
**Success criteria:**
- Drag-and-drop CSV file → loaded into DuckDB-WASM
- File picker alternative for CSV import
- Schema browser: column names, types, row count
- Data preview: first 100 rows in a scrollable table
- SQL cell type: write SQL, executes against DuckDB, returns table
- Column click → basic stats (min, max, mean, nulls, histogram)
- Emits `data:loaded` event on the event bus
- Handles large CSVs (streaming with Papa Parse, not loading all into memory)
- Unit tests for CSV parsing and DuckDB query execution

**Depends on:** Milestone 0 (event bus), Milestone 1 (cells for SQL)

### Milestone 4: Viz Module (2D Only)
**What:** Plotly.js charts rendered from cell output.
**Success criteria:**
- Viz panel renders Plotly.js charts
- Viz cell type: generates Plotly JSON spec → renders chart
- Supports: line, bar, scatter, histogram, heatmap (minimum 5 types)
- Charts are interactive (zoom, pan, hover tooltips)
- Listens to `data:loaded` events, suggests chart types
- Chart export: PNG, SVG
- No dependency on 3D (Three.js not needed yet)
- Unit tests for Plotly spec generation

**Depends on:** Milestone 0 (event bus), Milestone 3 (needs data to visualize)

### Milestone 5: MCP + Chat Module
**What:** LLM integration via MCP. Chat panel. LLM can create/edit cells.
**Success criteria:**
- MCP server exposes notebook tools (create_cell, edit_cell, query_data, etc.)
- MCP client connects to Claude API (Sonnet 4.5)
- MCP client connects to Ollama (Qwen 2.5 Coder 14B) for local use
- Chat panel: user types message → LLM responds
- LLM can read notebook state (cell list, data schema)
- LLM proposes cells → appear in notebook for user review
- Proposed cells have visual distinction (not auto-executed)
- User approves → cell executes. User rejects → cell removed.
- Conversation history managed with sliding window
- Settings: choose provider (Claude/Ollama/OpenAI), enter API key
- Integration test: send message → LLM creates a cell → cell appears

**Depends on:** Milestone 1 (notebook), Milestone 2 (runtime), Milestone 3 (data)

### Milestone 6: Yjs Concurrent Editing
**What:** User and LLM can edit the notebook simultaneously without conflicts.
**Success criteria:**
- Notebook document backed by Yjs CRDT
- User edits and LLM edits merge automatically (different cells)
- Same-cell edits: merged text with diff indicator
- Every edit tagged with author ("user" or "llm")
- Runtime snapshots before LLM multi-cell operations
- Snapshot rollback works in <100ms
- Stress test: rapid alternating user/LLM edits don't corrupt state

**Depends on:** Milestone 1, Milestone 5

### Milestone 7: 3D Visualization
**What:** Three.js viewport for 3D charts and scenes.
**Success criteria:**
- 3D viewport panel with orbit controls (rotate, zoom, pan)
- 3D scatter, surface, and bar chart types
- "Add Dimension" button on 2D charts → select Z column → 3D version
- WebGPU renderer (WebGL fallback for older hardware)
- Animated transitions between 2D and 3D (GSAP)
- Handles 100K+ data points at 30+ FPS

**Depends on:** Milestone 4 (2D viz must work first)

### Milestone 8: Animation & Time Scrubber
**What:** Animate parameters, scrub through time.
**Success criteria:**
- Timeline scrubber on any time-series visualization
- Parameter sweep: select variable, set range, animate
- Play/pause/speed controls
- Export: GIF, MP4
- Works in both 2D and 3D

**Depends on:** Milestone 4, Milestone 7

---

## Build Protocol: How Agents Build This Project

### Repository Structure

```
holy-alpha/
├── CLAUDE.md                    # Agent context (updated after every session)
├── AGENTS.md                    # Universal agent instructions
├── DECISIONS.md                 # Architecture decision log
├── BUILD_CONTEXT.md             # Current state, next steps, blockers
├── packages/
│   ├── electron-shell/          # Electron main process + IPC
│   ├── notebook/                # Notebook module (cells, editor)
│   ├── runtime/                 # Python execution + DAG engine
│   ├── data/                    # Data loading + DuckDB
│   ├── viz/                     # Visualization (Plotly + Three.js)
│   ├── chat/                    # Chat panel + MCP client
│   ├── mcp-server/              # MCP server (tools for LLM)
│   ├── event-bus/               # Shared event bus
│   └── shared/                  # Shared types, utilities
├── python/                      # Bundled Python runtime + packages
├── tests/
│   ├── unit/                    # Per-module unit tests
│   ├── integration/             # Cross-module integration tests
│   └── e2e/                     # End-to-end Electron tests
└── .github/
    └── workflows/               # CI: lint, typecheck, test
```

Monorepo with package-per-module. Each package has its own `package.json`,
`tsconfig.json`, and test suite. Agents work on separate packages in parallel.

### CLAUDE.md Protocol

CLAUDE.md is the living context file. Updated after EVERY build session.

**Structure:**
```markdown
# CLAUDE.md

## Project
Holy Alpha - AI-native data analysis notebook.
Electron + React + TypeScript. Monorepo with packages/.

## Commands
- `npm run dev` — Start Electron in dev mode
- `npm run test` — Run all unit tests
- `npm run test:integration` — Integration tests
- `npm run lint` — ESLint + Prettier
- `npm run typecheck` — TypeScript strict mode

## Architecture
- Decoupled modules communicate via event bus (packages/event-bus)
- Runtime is a separate Python child process (packages/runtime)
- MCP server exposes notebook tools (packages/mcp-server)
- No module should import directly from another module

## Current State
[Updated after each session]
- Milestone X: COMPLETE
- Milestone Y: IN PROGRESS — [what's done, what's left]
- Milestone Z: NOT STARTED

## Conventions
- All event names: `module:action` (e.g., `data:loaded`, `cell:executed`)
- All IPC messages: typed with TypeScript interfaces in packages/shared
- Tests required for every public function
- No `any` types

## Known Issues
[Updated after each session]
- [Issue description + workaround if any]

## Do NOT
- Import between modules directly (use event bus)
- Add dependencies without checking bundle size impact
- Skip tests
- Modify CLAUDE.md structure (only update content within sections)
```

### BUILD_CONTEXT.md Protocol

Updated at the END of every agent session:

```markdown
# BUILD_CONTEXT.md

## Last Updated
[Date + agent session ID]

## What Was Done This Session
- [Bullet list of completed work]
- [Files created/modified]
- [Tests added]

## Current Blockers
- [Any issues that need resolution]

## Next Steps (Priority Order)
1. [Most important next task]
2. [Second priority]
3. [Third priority]

## Decisions Made
- [Decision: rationale]

## Technical Debt Introduced
- [Shortcuts taken that need cleanup later]
```

### DECISIONS.md Protocol

Append-only log of architectural decisions:

```markdown
# DECISIONS.md

## 001 — Use Electron over Tauri
**Date:** 2026-02-14
**Context:** Need consistent cross-platform rendering for WebGPU/3D.
**Decision:** Electron (ships Chromium, consistent rendering).
**Alternatives:** Tauri (lighter but WebKitGTK unstable on Linux).
**Consequences:** Larger bundle (~80MB), higher memory, but predictable.

## 002 — DuckDB-WASM as query engine
...
```

### Agent Task Assignment Protocol

**Rule: One agent per module. No agent touches another agent's module.**

```
Agent A → packages/notebook + packages/shared (types)
Agent B → packages/data + packages/event-bus
Agent C → packages/viz
Agent D → packages/mcp-server + packages/chat
```

**Before starting:**
1. Read CLAUDE.md
2. Read BUILD_CONTEXT.md
3. Read DECISIONS.md
4. Read the milestone spec for your task
5. Check current test suite passes: `npm run test`

**After finishing:**
1. Run `npm run test` — all tests pass
2. Run `npm run lint` — no errors
3. Run `npm run typecheck` — no errors
4. Update BUILD_CONTEXT.md with what you did
5. Update CLAUDE.md if any new commands, conventions, or issues
6. Append to DECISIONS.md if you made architectural choices
7. Commit with descriptive message

### Evaluation Protocol

Every milestone has automated gates:

```
Gate 1: Type-check passes          (npm run typecheck)
Gate 2: Lint passes                (npm run lint)
Gate 3: Unit tests pass            (npm run test)
Gate 4: Integration tests pass     (npm run test:integration)
Gate 5: Manual review of UI/UX    (human)
Gate 6: Success criteria checklist (human verifies each criterion)
```

Agents self-evaluate against the success criteria checklist.
If any criterion is not met, the agent documents why and what's needed in BUILD_CONTEXT.md.

---

## Tech Stack (Final, Condensed)

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Shell | Electron 35+ | Desktop app, Chromium, cross-platform |
| UI | React 19, TypeScript | Frontend framework |
| Editor | Monaco Editor | Code cells |
| Math | KaTeX, math.js | Formula rendering + evaluation |
| CRDT | Yjs | Concurrent editing (user + LLM) |
| Query | DuckDB-WASM | In-browser SQL on CSV/Parquet |
| CSV | Papa Parse | Streaming CSV parser |
| 2D Viz | Plotly.js | Charts (40+ types) |
| 3D Viz | Three.js + WebGPU | 3D scenes, surfaces, animations |
| Animation | GSAP | Transitions |
| Compute | Python 3.12+ (child process) | numpy, pandas, scipy |
| AI | MCP SDK (@modelcontextprotocol/sdk) | LLM tool integration |
| LLM (prod) | Claude Sonnet 4.5 | Code gen, tool use |
| LLM (local) | Qwen 2.5 Coder 14B via Ollama | Offline, free |
| Storage | better-sqlite3 | Notebook metadata, saved state |
| Data Format | Apache Arrow (JS) | Cross-module data interchange |
| Bus | Custom EventEmitter | Module decoupling |
| Test | Vitest | Unit + integration tests |
| E2E Test | Playwright | Electron end-to-end |
| CI | GitHub Actions | Lint, typecheck, test on every push |

---

## What Ships When

| Milestone | What User Gets | Target |
|-----------|---------------|--------|
| M0 | Empty app launches | Week 1 |
| M1 | Can write and edit cells | Week 2-3 |
| M2 | Cells execute Python, reactive updates | Week 4-5 |
| M3 | Load CSV, browse data, write SQL | Week 6-7 |
| M4 | 2D charts from data | Week 8-9 |
| M5 | Chat with LLM, LLM creates cells | Week 10-12 |
| **v0.1 ALPHA RELEASE** | **CSV → analyze → visualize → chat** | **Week 12** |
| M6 | User + LLM edit simultaneously | Week 13-14 |
| M7 | 3D visualization | Week 15-17 |
| M8 | Animation + time scrubber | Week 18-20 |
| **v0.2 BETA RELEASE** | **Full 2D+3D viz with animation** | **Week 20** |

### Future Roadmap (Post v0.2)
- PostgreSQL / database connectivity
- Backtesting engine (VectorBT)
- Formula cells (KaTeX → executable)
- Education templates (math, probability)
- Plugin system (community tool adapters)
- Remote compute providers
- Collaborative editing (multi-user)
