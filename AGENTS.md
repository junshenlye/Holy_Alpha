# Agent Operating Protocol

This document defines the mandatory procedures every agent must follow when
working on Holy Alpha. No exceptions. No shortcuts.

---

## Before Starting Any Task

```
STEP 1: Read context files (MANDATORY)
  ├── Read CLAUDE.md           — project conventions and current state
  ├── Read BUILD_CONTEXT.md    — what was done last, blockers, next steps
  ├── Read DECISIONS.md        — architectural decisions already made
  └── Read the milestone spec  — success criteria for your task

STEP 2: Verify clean state (MANDATORY)
  ├── Run: npm run typecheck   — must pass
  ├── Run: npm run lint        — must pass
  └── Run: npm run test        — must pass
  If any fail: STOP. Fix or report in BUILD_CONTEXT.md before proceeding.

STEP 3: Create feature branch (MANDATORY)
  └── git checkout -b feature/<milestone>-<description>
      Example: feature/m1-notebook-core
```

⚠️ DO NOT write any code until all three steps are complete.

---

## While Working

### Commit Protocol

**Commit after every logical unit of work.** A logical unit is:
- One new function + its test
- One new component + its test
- One bug fix
- One configuration change

**NOT a logical unit:**
- "Made a bunch of changes" (too vague)
- An entire milestone in one commit (too large)
- Half a function (too small)

**Commit message format:**
```
<type>(<scope>): <description>

<body — what and why, not how>

Co-Authored-By: Claude <noreply@anthropic.com>
```

Types: feat, fix, refactor, test, docs, chore
Scope: package name (notebook, data, viz, chat, runtime, mcp-server, event-bus, shared)

**Before every commit:**
```bash
npm run lint && npm run typecheck && npm run test
```
If any fail: fix first, then commit. Never commit broken code.

### What to Commit vs What Not to Commit

**COMMIT:**
- Source code changes
- Test files
- Type definitions
- Configuration files (tsconfig, eslint, vite)
- BUILD_CONTEXT.md updates
- DECISIONS.md entries

**DO NOT COMMIT:**
- node_modules/
- dist/ or build/
- .env files with secrets
- Large binary files
- Auto-generated lock files (unless dependency change)

---

## Cross-Module Rules

### Never Import Across Packages

```typescript
// ❌ WRONG — direct import
import { Cell } from "../notebook/types";

// ✅ RIGHT — shared types
import { Cell } from "@holy-alpha/shared";

// ✅ RIGHT — event bus communication
import { bus } from "@holy-alpha/event-bus";
bus.emit("cell:created", { cellId, type, source });
```

### Data Passing

Between modules, data flows through:
1. **Event bus** — for notifications and loose coupling
2. **Shared types** — for TypeScript interfaces
3. **Apache Arrow** — for tabular data between runtime and renderer

Never pass raw objects between modules. Always use typed interfaces from `@holy-alpha/shared`.

---

## After Completing a Task

```
STEP 1: Verify all gates pass (MANDATORY)
  ├── npm run typecheck  — passes
  ├── npm run lint       — passes
  └── npm run test       — passes

STEP 2: Self-evaluate against success criteria (MANDATORY)
  For each criterion in the milestone spec:
    ├── MET     — describe how/where
    ├── NOT MET — explain why and what's needed
    └── PARTIAL — describe what's done and what remains

  If any criterion is NOT MET:
    └── Document in BUILD_CONTEXT.md under "Blockers"
        Do NOT mark the milestone as complete.

STEP 3: Update BUILD_CONTEXT.md (MANDATORY)
  ├── "What Was Done This Session" — bullet list of completed work
  ├── "Files Created/Modified"     — list every file touched
  ├── "Tests Added"                — list new test files
  ├── "Current Blockers"           — anything unresolved
  ├── "Next Steps"                 — priority-ordered next tasks
  └── "Technical Debt"             — shortcuts taken

STEP 4: Update CLAUDE.md if needed (MANDATORY)
  Only update these sections:
  ├── "Current State"  — milestone progress
  ├── "Known Issues"   — new issues discovered
  └── "Commands"       — if new commands were added

STEP 5: Update DECISIONS.md if applicable
  If you made an architectural choice, append:
  ├── Decision number (sequential)
  ├── Date
  ├── Context (why the decision was needed)
  ├── Decision (what was chosen)
  ├── Alternatives (what was considered)
  └── Consequences (tradeoffs accepted)

STEP 6: Final commit (MANDATORY)
  Commit BUILD_CONTEXT.md, DECISIONS.md, CLAUDE.md updates.
  Message: docs(<scope>): update build context after <what was done>
```

⚠️ DO NOT end a session without completing Steps 1-6.

---

## Error Handling

### If tests fail after your changes:
1. Read the error message carefully
2. Fix the failing test or the code that broke it
3. Run tests again
4. Only commit when all tests pass

### If you encounter a blocker:
1. Document it in BUILD_CONTEXT.md under "Current Blockers"
2. Describe: what you tried, why it failed, what you think the fix is
3. Do NOT attempt a hacky workaround
4. Do NOT delete or skip tests to make things pass

### If you're unsure about an architectural choice:
1. Check DECISIONS.md for prior decisions
2. If no prior decision covers it, document your reasoning
3. Choose the simpler option
4. Add it to DECISIONS.md
5. Note it in BUILD_CONTEXT.md so the next agent (or human) can review

---

## Module Ownership

When multiple agents work in parallel, each agent owns specific packages:

| Agent | Packages | Can Read | Cannot Modify |
|-------|----------|----------|---------------|
| A | notebook, shared | Any package | data, viz, chat, runtime |
| B | data, event-bus | Any package | notebook, viz, chat, runtime |
| C | viz | Any package | notebook, data, chat, runtime |
| D | chat, mcp-server | Any package | notebook, data, viz, runtime |
| E | runtime | Any package | notebook, data, viz, chat |

**Shared types (packages/shared):** Owned by whichever agent needs a new type.
Use a git worktree per agent to prevent conflicts.

---

## Quality Bar

Every piece of code must meet this bar before commit:

- [ ] TypeScript strict mode passes
- [ ] ESLint passes with zero warnings
- [ ] Every public function has a test
- [ ] Every public function has JSDoc
- [ ] No `any` types
- [ ] No `console.log` (use logger)
- [ ] No hardcoded strings (use constants from shared)
- [ ] Event names follow `module:action` convention
- [ ] No cross-package imports (use event bus or shared)
