# Holy Alpha

AI-native data analysis notebook. Electron + React + TypeScript monorepo.
Decoupled modules communicate via event bus. Runtime is a separate process.

## Commands
```
npm run dev              # Start Electron dev mode
npm run build            # Production build
npm run test             # Run all unit tests (Vitest)
npm run test:integration # Integration tests
npm run lint             # ESLint + Prettier
npm run typecheck        # TypeScript strict mode (must pass)
```

## Architecture

Monorepo with independent packages. **No package imports another directly.**
All cross-module communication goes through `packages/event-bus`.

```
packages/
  electron-shell/    # Electron main process, IPC, window management
  notebook/          # Cell editor (Monaco), cell CRUD, file persistence
  runtime/           # Python child process, DAG engine, variable store
  data/              # CSV loading (Papa Parse), DuckDB-WASM queries
  viz/               # Plotly.js (2D), Three.js (3D), chart rendering
  chat/              # Chat panel UI, MCP client, provider switching
  mcp-server/        # MCP tools exposed to LLMs (create_cell, query_data, etc.)
  event-bus/         # Typed EventEmitter, all event definitions
  shared/            # TypeScript interfaces, constants, utilities
```

## Event Bus Convention
All events: `module:action` format.
```typescript
"data:loaded"      // { tableId, schema, rowCount }
"cell:created"     // { cellId, type, source }
"cell:executed"    // { cellId, output, error }
"cell:updated"     // { cellId, source, author: "user" | "llm" }
"viz:rendered"     // { cellId, chartType }
"runtime:ready"    // { pythonVersion }
"runtime:error"    // { cellId, traceback }
"chat:message"     // { role, content }
"chat:proposal"    // { cellId, source, explanation }
```

## Code Conventions
- TypeScript strict mode. No `any` types. Use `unknown` if type is truly dynamic.
- Functional components only (React). No class components.
- Named exports only (no default exports).
- All public functions must have JSDoc with @param and @returns.
- Test every public function. Colocate tests: `foo.ts` → `foo.test.ts`.
- Use Zod for runtime validation at module boundaries.

## Git Workflow
- Conventional Commits: `feat(notebook): add cell reordering`
- One logical change per commit. Atomic commits, not WIP.
- Feature branches: `feature/m1-notebook-core`
- Co-Authored-By: Claude <noreply@anthropic.com> on AI-generated commits.
- Run `npm run lint && npm run typecheck && npm run test` before committing.

## Current State
- Project: NOT STARTED
- Next milestone: M0 (Project Scaffolding)

## Known Issues
(none yet)

## Do NOT
- Import between packages directly (use event bus or shared types)
- Add dependencies without checking bundle size (`npx cost-of-modules`)
- Skip tests for any public function
- Modify this file's structure (only update content within sections)
- Use `console.log` for debugging (use a logger utility)
- Commit with failing tests or type errors
