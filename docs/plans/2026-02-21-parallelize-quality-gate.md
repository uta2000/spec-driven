# Parallelize Quality Gate Checks — Design Document

**Date:** 2026-02-21
**Status:** Draft
**Issue:** #42

## Overview

The quality gate hook (`hooks/scripts/quality-gate.js`) runs TypeScript, lint, type-sync, and test checks sequentially using `execSync`. Since the first three checks are independent, they can run in parallel using `Promise.allSettled` with promisified `exec`, reducing wall-clock time from ~43s to ~25s per verification pass (bounded by the slowest check). Tests remain sequential because they depend on typecheck passing.

## User Flow

### Step 1 — Session ends, hook triggers
The `Stop` hook invokes `quality-gate.js`. The script detects available tooling.

### Step 2 — Independent checks run in parallel
TypeScript (`tsc --noEmit`), lint (`npm run lint` or direct linter), and type-sync all execute concurrently via `Promise.allSettled`.

### Step 3 — Tests run conditionally
If the TypeScript check found no type errors, tests run. If types are broken, tests are skipped (no point running tests against broken types).

### Step 4 — Results reported
All failures and warnings collected and reported as before — `BLOCK:` for failures, `console.error` for warnings.

## Pipeline / Architecture

### Current flow (sequential)
```
checkTypeScript() → checkLint() → checkTypeSync() → checkTests()
~25s              ~10s           ~8s               ~20s = ~63s total
```

### New flow (parallel + conditional)
```
┌─ checkTypeScript() ─┐
├─ checkLint()         ├─ await all → checkTests() (if typecheck passed)
└─ checkTypeSync()     ┘
~25s (bounded by slowest)        ~20s = ~45s total
```

### Key technical decisions

1. **`util.promisify(exec)`** over `execFile` — current commands use shell features (`npx`, `npm run`), and `exec` spawns a shell. The promisified error object includes `.stdout`/`.stderr`, matching the existing `execOutput(e)` helper.

2. **`Promise.allSettled`** over `Promise.all` — ensures all checks complete even if one throws unexpectedly. Each settled result is inspected: `fulfilled` means the check ran (it may have pushed to `failures[]` internally); `rejected` means an unexpected crash (pushed to `warnings[]`).

3. **Module-level `failures`/`warnings` arrays** — kept as shared mutable state. Safe in Node.js single-threaded event loop: async tasks interleave at await points but never run simultaneously, so array pushes never race.

4. **Tests gated on typecheck** — new behavior. If `checkTypeScript()` pushed any entries to `failures[]`, `checkTests()` is skipped. This avoids noisy test failures that are just symptoms of type errors.

5. **`async function main()`** — wraps the top-level orchestration. Called with `.catch()` to handle unexpected errors and ensure `process.exit(0)` always runs (hook protocol: exit 0, communicate via output).

### Changes to SKILL.md

Add a parallelization note to Code Review Pipeline Phase 4 (re-verify loop, line ~718) instructing the LLM to dispatch typecheck, lint, and type-sync as parallel Bash tool calls in a single message, running tests only after typecheck passes.

## Scope

**Included:**
- Convert `execSync` → async `exec` in all check functions
- Wrap top-level in `async main()` with `Promise.allSettled`
- Gate test execution on typecheck success
- Add parallelization guidance to SKILL.md Phase 4

**Excluded:**
- No changes to `lint-file.js` (per-file hook, already fast)
- No changes to hook configuration (`hooks.json`)
- No changes to the check detection logic itself (same tooling detection)
- No new dependencies (uses built-in `util.promisify`)
