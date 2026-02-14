# Holy Alpha - Design Document v3

## Vision

Holy Alpha is an **orchestration layer** disguised as a notebook. It doesn't try to
do everything itself. It connects open-source tools together, with an LLM as the
bridge that knows which tool to use for which job.

The notebook is a workspace — not a prescriptive UI. Users see code, data, and
visualizations in panels they control. The LLM and user collaborate on the same
notebook with proper state management so they don't collide.

---

## Core Architecture: The Orchestration Layer

Holy Alpha is NOT a monolithic app. It is three things:

```
1. A NOTEBOOK        — where code, data, and results live (like Jupyter)
2. A RUNTIME         — separated from the notebook, manages execution state
3. AN ORCHESTRATOR   — LLM bridges gaps between open-source tools
```

### The Key Insight: Tool Bridging

No single open-source tool does everything. But the LLM knows the landscape:

```
User: "Model this data and visualize it in 3D"

LLM thinks:
  - Data is in a CSV → use DuckDB to query it
  - Modeling needs curve fitting → use scipy.optimize
  - 3D visualization → scipy can't do that → use Three.js
  - Data format bridge: scipy outputs numpy arrays → serialize to Arrow → Three.js reads it

LLM generates:
  Cell 1: DuckDB SQL query (data extraction)
  Cell 2: scipy curve_fit (modeling)
  Cell 3: Three.js scene (visualization, fed by Cell 2 output)
```

The user sees the code for each step. They can modify any cell. The LLM picked the
tools, but the user owns the notebook.

### Tool Registry

Every tool Holy Alpha can use is registered with a standard interface:

```typescript
interface Tool {
  name: string              // "duckdb", "scipy", "threejs", "plotly", "vectorbt"
  capabilities: string[]    // ["sql-query", "csv-parse", "parquet-read"]
  inputFormats: string[]    // ["csv", "arrow", "json", "numpy-array"]
  outputFormats: string[]   // ["arrow", "json", "numpy-array", "plotly-figure"]
  runtime: "python" | "javascript" | "wasm" | "native"
  description: string       // Human-readable, LLM uses this for tool selection
}
```

When the LLM needs to solve a task, it queries the registry:
- "I need to fit a curve" → scipy, sklearn, numpy (all can do it)
- "I need 3D rendering" → Three.js (only option for high-quality 3D)
- "I need SQL on a CSV" → DuckDB-WASM (fast, in-browser)

When one tool has a limitation, the LLM finds another tool to fill the gap.
This is extensible — community can add new tools to the registry.

### Data Flow Between Tools (Apache Arrow as Lingua Franca)

The hard problem: scipy outputs numpy arrays, Three.js needs JavaScript objects,
DuckDB outputs columnar data. How do they talk?

**Apache Arrow** is the bridge:

```
DuckDB (SQL result)
    → Arrow Table (columnar, zero-copy)
        → Python runtime reads as numpy/pandas (zero-copy via pyarrow)
        → JavaScript runtime reads via Arrow JS bindings
        → Three.js geometry built from Arrow buffers

scipy (numpy array output)
    → Arrow Array (zero-copy wrap)
        → Serialized as Arrow IPC
        → Sent to renderer process
        → Three.js reads and visualizes
```

Arrow is the universal format. Every tool adapter converts to/from Arrow.
This means adding a new tool only requires writing one adapter.

---

## The Runtime: Separated, Stateful, Collaborative

### Problem: Jupyter's Broken State Model

Jupyter's state is a mess because:
- Kernel holds all state in memory (variables, imports, side effects)
- Executing cells out of order creates hidden state
- Deleting a cell doesn't delete its variables from the kernel
- Only 4% of GitHub notebooks reproduce when re-run
- If user edits cell 3 while LLM edits cell 7, nobody knows what state is valid

### Solution: DAG-Based Reactive Runtime (Separated from Notebook)

The runtime is a **separate process** that maintains a dependency graph:

```
┌─────────────────────────────────────────────────────────┐
│                  NOTEBOOK (UI Layer)                     │
│                                                         │
│  The user and LLM both edit cells here.                 │
│  This is just TEXT — no execution happens here.          │
│  Changes are tracked via Yjs CRDT (conflict-free).      │
│                                                         │
└────────────────────────┬────────────────────────────────┘
                         │ Messages (execute, snapshot, etc.)
                         │
┌────────────────────────▼────────────────────────────────┐
│                  RUNTIME (Execution Layer)               │
│                                                         │
│  Maintains:                                             │
│  - Dependency DAG (which cells depend on which)         │
│  - Variable store (current value of every variable)     │
│  - Execution history (what ran, in what order)          │
│  - Snapshots (can roll back to any point)               │
│                                                         │
│  Rules:                                                 │
│  - When cell X changes, re-run X and all dependents     │
│  - Execution order is DETERMINISTIC (from DAG, not      │
│    cell position)                                       │
│  - Variables are scoped — deleting a cell deletes its   │
│    variables                                            │
│  - State is always reproducible from the cell code      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### How the DAG Works

```python
# Cell A
data = load_csv("sales.csv")

# Cell B (depends on A)
monthly = data.groupby("month").sum()

# Cell C (depends on B)
plot(monthly)

# Cell D (depends on A, independent of B and C)
yearly = data.groupby("year").sum()
```

The runtime builds:
```
    A (data)
   / \
  B   D
  |
  C
```

- Edit Cell A → re-runs A, B, C, D (all depend on A)
- Edit Cell B → re-runs B, C (D is unaffected)
- Edit Cell D → re-runs D only
- Delete Cell B → removes `monthly` variable, Cell C shows error (dependency broken)

This is deterministic. No hidden state. No execution order surprises.

### Concurrent User + LLM Editing

The hard problem: user types in cell 3 while LLM modifies cell 7.

**Solution: CRDT Notebook + Message-Passing Runtime**

```
┌──────────────────────────────────────────────┐
│              Yjs CRDT Document                │
│                                              │
│  Both user and LLM edit the same CRDT.       │
│  Yjs handles text-level conflicts             │
│  automatically (character-level merge).       │
│                                              │
│  Each cell is a Y.Text with metadata.         │
│  Cell order is a Y.Array.                     │
│  Every edit has an AUTHOR tag: "user" | "llm" │
│                                              │
└──────────────┬───────────────────────────────┘
               │
               │  On change: debounced (300ms)
               │
┌──────────────▼───────────────────────────────┐
│           Runtime Execution Queue              │
│                                              │
│  Changes arrive as messages:                  │
│  { cellId, newCode, author, timestamp }       │
│                                              │
│  Conflict rules:                              │
│  1. If user and LLM edit DIFFERENT cells:     │
│     → Both accepted, re-run affected branches │
│                                              │
│  2. If user and LLM edit the SAME cell:       │
│     → Yjs merges the text (CRDT)              │
│     → Runtime flags it for user review         │
│     → Shows diff: "LLM changed X, you         │
│       changed Y — keep both / pick one"        │
│                                              │
│  3. If LLM edit breaks a dependency:           │
│     → Runtime detects via DAG                  │
│     → Rolls back LLM edit                      │
│     → Notifies LLM: "your edit to cell 7       │
│       broke cell 9's dependency on `result`"   │
│                                              │
└──────────────────────────────────────────────┘
```

**Why Yjs?**
- Production-proven (JupyterLab uses it for real-time collaboration)
- Character-level conflict resolution without a central server
- Every edit converges to the same state regardless of order
- Lightweight, well-maintained, MIT licensed

**Why NOT just git branches for LLM edits?**
Git branches are too heavy for character-level edits during active collaboration.
Git is for versioning notebooks (save/load), not for real-time concurrent editing.
Yjs handles the real-time layer; git handles the persistence layer.

### Runtime Snapshots

Before any LLM operation that touches multiple cells:

```
1. Runtime takes snapshot (all variables + DAG state)
2. LLM edits proceed
3. If edits succeed → snapshot discarded
4. If edits fail or user rejects → restore snapshot in <100ms
```

This is how Databricks handles serverless notebook state restoration.
Snapshots are in-memory (fast) with periodic disk persistence.

---

## The Workspace: Panels, Not Prescribed Levels

### No "Level 1 / Level 2 / Level 3"

You're right — prescribing experience levels is backwards. Instead:

**The workspace has functional panels. Users arrange them how they want.**

```
┌─────────────────────────────────────────────────────────────┐
│  Holy Alpha Workspace                              [Layout] │
│                                                             │
│  ┌─────────────────────┐  ┌──────────────────────────────┐ │
│  │                     │  │                              │ │
│  │   CHAT              │  │   NOTEBOOK                   │ │
│  │                     │  │                              │ │
│  │   Conversation with │  │   Cells: code, markdown,     │ │
│  │   LLM. Ask          │  │   formulas. Both user and    │ │
│  │   questions, get    │  │   LLM can edit. Standard     │ │
│  │   suggestions.      │  │   Jupyter-style.             │ │
│  │                     │  │                              │ │
│  │   LLM can propose   │  │   Full code visible.         │ │
│  │   cells → they      │  │   No hiding.                 │ │
│  │   appear in the     │  │   User controls everything.  │ │
│  │   notebook.         │  │                              │ │
│  │                     │  │                              │ │
│  └─────────────────────┘  └──────────────────────────────┘ │
│  ┌─────────────────────┐  ┌──────────────────────────────┐ │
│  │                     │  │                              │ │
│  │   DATA              │  │   VISUALIZATION              │ │
│  │                     │  │                              │ │
│  │   Live view of      │  │   Charts, 3D scenes,         │ │
│  │   loaded data.      │  │   animations.                │ │
│  │   Tables, schema,   │  │                              │ │
│  │   row counts.       │  │   Output of viz cells        │ │
│  │                     │  │   rendered here.              │ │
│  │   Verify your data  │  │                              │ │
│  │   is correct before │  │   2D (Plotly) and 3D         │ │
│  │   modeling.         │  │   (Three.js) in same panel.  │ │
│  │                     │  │                              │ │
│  └─────────────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Four panels, all visible, user resizes/rearranges:**

1. **Chat** — Conversation with LLM. The LLM reads the notebook context and can
   propose new cells, explain existing ones, or answer questions. Chat messages
   can be "pinned" to become notebook cells.

2. **Notebook** — Standard code cells, like Jupyter. Markdown, Python, SQL,
   formula cells. Both user and LLM have access. Full code always visible.
   This is the source of truth.

3. **Data** — Live view of all loaded datasets. Tables with virtual scrolling.
   Schema browser for database connections. This is where you VERIFY the data
   before doing anything with it. Click a column → see distribution.
   This panel is what makes it easy for non-technical users.

4. **Visualization** — Where charts and 3D scenes render. Output from viz cells
   appears here. Interactive (zoom, rotate, scrub). Can be expanded fullscreen.

**Users can:**
- Collapse any panel
- Rearrange panels (drag to reorder)
- Focus a single panel (fullscreen mode)
- Open multiple viz panels (compare charts side by side)
- A kid might use Chat + Visualization mostly
- A quant might use Notebook + Data mostly
- Nobody is forced into a "level"

---

## How the LLM Orchestrates Tools

### The MCP Tool Interface

Holy Alpha is an MCP client AND server:

**As MCP Client** (connects to LLMs):
- Sends notebook context (cell code, data schemas, variable values) to LLM
- Receives tool calls back (create cell, edit cell, run query, generate viz)

**As MCP Server** (exposes notebook as tools):
- External LLMs can connect and manipulate the notebook
- Tools exposed: `create_cell`, `edit_cell`, `run_cell`, `query_data`,
  `create_visualization`, `read_variable`, `list_datasets`

### Tool Selection Flow

```
User: "Fit a polynomial to this data and show me the curve"

LLM receives:
  - Available tools: [duckdb, numpy, scipy, sklearn, plotly, threejs, ...]
  - Current data: schema of loaded dataset
  - Current cells: existing code in notebook

LLM decides:
  1. Data is already loaded (DuckDB) → extract x, y columns → SQL cell
  2. Polynomial fitting → scipy.optimize.curve_fit OR numpy.polyfit
     → picks numpy.polyfit (simpler for polynomials)
  3. Visualization → 2D line + scatter → Plotly (good enough, no need for Three.js)
  4. Data bridge: DuckDB result → Arrow → numpy array → Plotly trace

LLM generates 3 cells:
  Cell 1: SQL query to extract x, y
  Cell 2: numpy.polyfit with degree slider
  Cell 3: Plotly scatter + fitted line

User sees all 3 cells in the notebook.
User can edit any of them.
Runtime tracks dependencies: Cell 3 depends on Cell 2 depends on Cell 1.
```

### When Tools Have Gaps

```
User: "Show me this surface plot with lighting and shadows"

LLM knows:
  - Plotly 3D can do surface plots but lighting/shadow control is limited
  - Three.js has full lighting, materials, shadows
  - Gap: Plotly can't do it → bridge to Three.js

LLM generates:
  Cell 1: Compute surface mesh data (numpy)
  Cell 2: Three.js scene with:
    - MeshStandardMaterial (physically-based rendering)
    - DirectionalLight with shadow mapping
    - OrbitControls for user interaction

Data bridge: numpy mesh → Arrow serialization → Three.js BufferGeometry
```

The user doesn't need to know which tool the LLM picked. But they CAN see it
in the code. And they CAN change it. "Use Plotly instead" → LLM regenerates.

### Extensible Tool Registry

New tools are added as adapters:

```typescript
// Example: adding a new tool
const vectorbtAdapter: ToolAdapter = {
  name: "vectorbt",
  capabilities: ["backtest", "portfolio-optimization", "technical-indicators"],
  runtime: "python",
  inputFormats: ["pandas-dataframe", "numpy-array"],
  outputFormats: ["pandas-dataframe", "plotly-figure"],

  // What the LLM sees when deciding which tool to use
  description: `VectorBT is a high-performance backtesting library.
    Use it when: user wants to backtest a trading strategy, calculate
    technical indicators, or optimize portfolio parameters.
    Do NOT use it for: general data analysis, visualization, or ML models.
    Strengths: Can run 1M orders in 70-100ms. Vectorized operations.
    Limitations: Trading strategies only. Not a general-purpose tool.`,

  // How to convert data to/from this tool's format
  toArrow: (pandasDf) => { /* pandas → Arrow */ },
  fromArrow: (arrowTable) => { /* Arrow → pandas */ },
}

// Register it
toolRegistry.register(vectorbtAdapter);
```

Community can publish tool adapters as npm/pip packages.
LLM automatically discovers new tools when they're registered.

---

## Data Connectivity (Detailed)

### Architecture

```
┌──────────────────────────────────────────┐
│          Electron Renderer               │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │         DuckDB-WASM                │  │
│  │  (in-browser query engine)         │  │
│  │                                    │  │
│  │  All data ends up here for SQL.    │  │
│  │  Query CSV, Parquet, JSON, Arrow.  │  │
│  │  6ms warm / 40ms cold queries.     │  │
│  └──────────────┬─────────────────────┘  │
│                 │                         │
└─────────────────┼────────────────────────┘
                  │ IPC (Arrow format)
┌─────────────────┼────────────────────────┐
│  Electron Main  │                        │
│                 │                         │
│  ┌──────────────▼─────────────────────┐  │
│  │       Data Source Manager          │  │
│  │                                    │  │
│  │  ┌──────┐ ┌──────┐ ┌──────────┐   │  │
│  │  │ CSV  │ │ JSON │ │ Clipboard│   │  │
│  │  │      │ │      │ │  Paste   │   │  │
│  │  └──────┘ └──────┘ └──────────┘   │  │
│  │  ┌──────┐ ┌──────┐ ┌──────────┐   │  │
│  │  │Parq- │ │Post- │ │ REST API │   │  │
│  │  │ uet  │ │greSQL│ │ (future) │   │  │
│  │  └──────┘ └──────┘ └──────────┘   │  │
│  └────────────────────────────────────┘  │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │      Python Child Process          │  │
│  │  (heavy compute, ML, backtesting)  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### Data Source Details

**CSV (Phase 1):**
- Drag-and-drop or file picker
- Papa Parse for streaming (handles GB-sized files, row-by-row)
- Auto-detect delimiters, headers, types
- Loaded into DuckDB-WASM as a table
- User immediately sees: row count, columns, types, preview

**PostgreSQL (Phase 2):**
- Connection form in Data panel (host, port, db, user, password)
- `node-postgres` in Electron main process (not browser — browser can't do TCP)
- Schema browser: databases → tables → columns with types
- Click table → preview 100 rows
- Run SQL in notebook → results come back as Arrow tables
- Saved connections stored securely (Electron safeStorage / OS keychain)

**Parquet (Phase 2):**
- Drag-and-drop .parquet files
- DuckDB-WASM reads natively with column pruning + predicate pushdown
- Can also query REMOTE Parquet over HTTP (DuckDB-WASM range reads)
- Only downloads the bytes it needs — enables querying terabytes from a laptop

**TimescaleDB (Phase 3):**
- Same connector as PostgreSQL (TimescaleDB is a PG extension)
- Hypertable-aware schema browser
- Time-series specific query templates

**Future:**
- SQLite files (drag and drop)
- Excel/xlsx (via SheetJS)
- REST APIs (JSON endpoints)
- InfluxDB, QuestDB (HTTP APIs)
- Cloud storage (S3/GCS Parquet files via DuckDB httpfs)

### The Verify-First Pattern

Critical for non-technical users. Before ANY analysis:

```
1. Load data → Data panel shows table
2. User SEES the data (columns, types, sample rows, distributions)
3. User confirms: "This looks right"
4. THEN analysis begins

Not:
1. Load data → black box → here's your result → was the data even correct?
```

The Data panel always shows:
- Row count and column count
- Column types (string, number, date, etc.)
- Null count per column
- Basic stats (min, max, mean for numbers)
- Click a column → histogram/distribution
- Search/filter rows

---

## Computation Strategy

### Light vs Heavy Compute

**Light (DuckDB-WASM, in-browser):**
- SQL queries on loaded data
- Aggregations, joins, window functions
- Fast: 6ms typical query
- No Python needed
- Good for: data exploration, filtering, grouping

**Heavy (Python child process):**
- numpy, pandas, scipy, sklearn, vectorbt
- Full Python with all packages
- Spawned by Electron main process
- Communicates via IPC (Arrow serialization)
- Good for: modeling, ML, backtesting, optimization

**LLM decides which to use:**
- Simple aggregation → DuckDB SQL cell
- Curve fitting → Python (scipy) cell
- Backtest → Python (vectorbt) cell
- The user sees the code either way

### Python Bundling

For users without Python installed:

**Ship a bundled Python runtime** (~50MB compressed, included in installer):
- Minimal CPython distribution
- Pre-installed: numpy, pandas, scipy, scikit-learn, pyarrow
- User never sees it — it's an implementation detail
- Similar to how VS Code bundles Node.js

**For users who HAVE Python:**
- Detect system Python
- Option to use their environment (for custom packages)
- Useful for quants who need PyTorch, TensorFlow, etc.

### Future: Remote Compute

The architecture supports this by design:
- The IPC protocol between notebook and runtime is serializable
- A "compute provider" interface abstracts where code runs
- Local Python → same interface as → remote GPU server
- Future: plug in cloud GPU for heavy ML training
- Future: shared compute pools for teams
- But LOCAL must work well enough for 90% of use cases first

---

## Visualization Architecture

### Why Two Renderers

**Plotly.js (2D + basic 3D):**
- 40+ chart types
- Interactive by default
- Declarative JSON spec (ideal for LLM generation)
- Fast to render, good for data exploration
- Used for: scatter, line, bar, heatmap, histogram, box, etc.

**Three.js + WebGPU (advanced 3D):**
- Full 3D scene graph with lighting, materials, cameras
- WebGPU backend: 10x faster rendering, 150x faster compute
- WebGL fallback for older hardware
- Used for: 3D surfaces with shadows, animated scenes, large point clouds,
  custom visualizations that Plotly can't handle

**The LLM picks the right renderer:**
- "Plot a bar chart" → Plotly
- "Show a 3D surface with realistic lighting" → Three.js
- "Scatter plot with 1M points" → Plotly WebGL or Three.js (both can handle it)
- User can override: "Use Three.js for this instead"

### 2D → 3D Promotion (Dimension Ladder)

Any 2D chart can gain a dimension:
- Scatter (x,y) → "Add Z axis" → select column → Scatter3D
- Line chart → "Extrude over time" → Surface
- Heatmap → "Elevate" → 3D terrain

This generates a new cell with the 3D code. The user sees and owns the code.
It's not magic — it's a cell that the LLM wrote, that you can edit.

### Animation

- Timeline scrubber on any time-dependent visualization
- Parameter sweep: animate a variable over a range
- Each frame is a computation — the runtime handles it
- Export: GIF, MP4, WebM
- Use cases: portfolio evolution, gradient descent, Monte Carlo simulations

---

## Education Use Cases

### How It Works for a Kid Learning Quadratics

```
Student in Chat: "What does y = x² look like?"

LLM generates cell:
  - Code: plotly.express.line with x range and y = x²
  - Adds slider for coefficient: y = a·x²

Student sees: parabola in Visualization panel
Student drags slider → parabola stretches/compresses
Student in Chat: "What happens with y = x² + 3?"

LLM generates new cell:
  - Previous parabola + shifted version
  - Explains: "+3 shifts the curve up by 3 units"

Student sees: both curves overlaid, with annotation
```

The student uses Chat + Visualization panels mostly.
The Notebook panel shows the code (they can look if curious).
Nothing is hidden. Nothing is forced.

### How It Works for a Trader Backtesting

```
Trader connects PostgreSQL (historical prices)
Data panel shows: tables, columns, date ranges

Trader in Chat: "Test a simple moving average crossover on AAPL"

LLM generates cells:
  Cell 1: SQL to fetch AAPL close prices from DB
  Cell 2: vectorbt SMA crossover strategy (code visible)
  Cell 3: Equity curve (Plotly), drawdown chart, stats table

Trader reviews Cell 2 code → changes SMA periods from 20/50 to 10/30
Runtime re-runs Cell 2 → Cell 3 updates automatically (DAG dependency)

Trader in Chat: "Now optimize the SMA periods"
LLM generates Cell 4: parameter grid search
  → Results as 3D surface (Sharpe ratio vs fast_period vs slow_period)
  → Three.js because it needs proper lighting for readability
```

Same workspace. Different panels emphasized. No level system.

---

## Technical Stack (Final)

### Electron App
```
Electron 35+                 — Desktop shell (Chromium, consistent rendering)
React 19 + TypeScript        — UI framework
Yjs                          — CRDT for concurrent notebook editing (user + LLM)
Monaco Editor                — Code cells (same engine as VS Code)
```

### Query & Data
```
DuckDB-WASM                  — In-browser SQL engine (CSV, Parquet, JSON, Arrow)
Papa Parse                   — CSV streaming parser
Apache Arrow (JS)            — Universal data interchange format
node-postgres (pg)           — PostgreSQL connector (Electron main process)
better-sqlite3               — Local storage (notebook metadata, saved connections)
```

### Visualization
```
Plotly.js                    — 2D charts + basic 3D (declarative, LLM-friendly)
Three.js                     — Advanced 3D (WebGPU primary, WebGL fallback)
GSAP                         — Animation transitions
```

### Compute
```
Python 3.12+ (bundled)       — Native runtime for heavy compute
numpy, pandas, scipy         — Core scientific computing
scikit-learn                 — ML models
vectorbt                     — Backtesting
pyarrow                      — Arrow bridge between Python and JS
```

### AI
```
MCP Client                   — Connects to Claude API, OpenAI API, Ollama, LM Studio
MCP Server                   — Exposes notebook tools to external LLMs
Tool Registry                — Adapters for each open-source tool
```

### Formula
```
KaTeX                        — LaTeX formula rendering
math.js                      — Formula parsing and evaluation
```

### Runtime
```
Custom DAG Engine            — Reactive dependency tracking
Yjs                          — Concurrent editing (CRDT)
WebWorkers                   — Non-blocking execution
IPC (Arrow serialization)    — Communication between renderer and main process
```

---

## What Holy Alpha IS and IS NOT

**IS:**
- An orchestration layer that bridges open-source tools
- A Jupyter-style notebook with a proper stateful runtime
- A workspace with functional panels (chat, code, data, viz)
- A system where LLM and user collaborate on the same document safely
- A way to make data analysis accessible without removing transparency
- Extensible via tool adapters (community can add tools)

**IS NOT:**
- A black box that hides computation
- A prescriptive UI that dictates user experience
- A replacement for any single tool (it uses them all)
- A cloud platform (desktop-first, your data stays local)
- A live trading platform (analysis and backtesting only)
- A monolithic app that tries to do everything itself

---

## Open Questions

1. **Python bundling**: Embed a minimal Python in the Electron app? Or require
   system Python? Leaning toward bundling for zero-friction UX.

2. **MCP server marketplace**: Should we host a registry of community tool
   adapters? Or just use npm/pip packages?

3. **Notebook file format**: .ipynb (Jupyter compatible) or custom .holy format?
   Tradeoff: compatibility vs better features.

4. **Offline LLM**: Bundle a small model via Ollama for offline use?
   Or require API connection for v1?

5. **Collaboration**: Multi-user editing (remote) is technically possible with Yjs.
   But is it a v1 feature or future? Leaning toward future.
