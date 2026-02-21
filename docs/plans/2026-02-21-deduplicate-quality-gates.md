# Deduplicate Quality Gate Runs — Design Document

**Date:** 2026-02-21
**Status:** Draft
**Issue:** #36

## Overview

Quality gates (typecheck, lint, type-sync, tests) run redundantly across three lifecycle phases with no coordination: Code Review Phase 4 re-verify (up to 3 iterations), Final Verification, and the Stop hook. Session analysis found 8 typecheck runs in a 26-minute session, with 5-6 being redundant. This change adds commit-hash based skip logic at all three dedup points, eliminating redundant runs when code hasn't changed since the last successful pass.

## Example

**Before:** Code review Phase 4 passes all quality gates. Final Verification runs them again (unchanged code). Stop hook runs them a third time. Total: 3 redundant passes of typecheck (~25s each = 75s wasted).

**After:** Code review Phase 4 passes. Final Verification checks `git status --porcelain` — working tree clean, skips quality gates, only runs `verify-acceptance-criteria`. Final Verification writes HEAD hash to `.git/feature-flow-verified`. Stop hook reads marker, HEAD matches, working tree clean — skips all checks. Total: 0 redundant passes.

## Pipeline Architecture

Three dedup points, one mechanism:

| Phase | Location | Dedup Check | Skip When |
|-------|----------|-------------|-----------|
| Code Review Phase 4 (re-verify loop) | `skills/start-feature/SKILL.md` | `git diff --stat` between iterations | No files changed since last iteration's pass |
| Final Verification | `skills/start-feature/SKILL.md` | `git status --porcelain` | Phase 4 passed quality gates and working tree is clean |
| Stop hook | `hooks/scripts/quality-gate.js` | Marker file + HEAD hash + `git diff HEAD` | Marker matches current HEAD and no dirty tracked files |

**Marker file:** `.git/feature-flow-verified` — contains the HEAD commit hash written after Final Verification passes. Located in `.git/` so it is never tracked by git.

**Marker lifecycle:**
1. Written by the LLM during the Final Verification step: `git rev-parse HEAD > .git/feature-flow-verified`
2. Read by `quality-gate.js` at the top of `main()`
3. Naturally invalidated when HEAD changes (new commits) or files are modified (dirty working tree)

## Changes

### 1. Code Review Phase 4 — skip unchanged iterations

Add a "skip if clean" instruction to SKILL.md before the re-verify quality checks. Between iterations of the fix-verify loop, check `git diff --stat` — if no source files changed since the last successful pass, skip the quality gate re-run and announce the skip.

This targets the 3-iteration loop where fixes may not change code (e.g., a Minor finding is logged but not fixed, triggering a re-verify with identical files).

### 2. Final Verification — skip when Phase 4 already passed

Add conditional skip logic to the Final Verification step description. If Code Review Phase 4's last iteration passed all quality gates AND `git status --porcelain` returns empty (no modifications since), skip the `verification-before-completion` quality gates. Only run `verify-acceptance-criteria`.

The rationale: Phase 4 already ran typecheck, lint, and tests. If nothing changed between Phase 4 completion and Final Verification, re-running them is pure waste.

### 3. Final Verification — write marker file

After all checks pass (acceptance criteria + any quality gates that ran), write the HEAD commit hash to `.git/feature-flow-verified`. This is a single Bash command the LLM executes inline.

### 4. Stop hook — early-exit with marker check

Add early-exit logic to the top of `main()` in `quality-gate.js`. Read `.git/feature-flow-verified`, compare its contents with `git rev-parse HEAD`, and check `git diff HEAD --name-only` for dirty tracked files. If the marker matches and working tree is clean, skip all checks.

Uses `execSync` (already imported) for the git commands — they complete in <10ms and must finish before the skip decision.

**Safety guarantees:**
- Missing marker → checks run (fail-open, covers non-lifecycle sessions)
- Corrupted/unreadable marker → checks run (catch block falls through)
- HEAD changed since marker → checks run (new commits invalidate)
- Dirty working tree → checks run (uncommitted changes need checking)
- `execSync` failure → checks run (catch block falls through)

## Scope

**Included:**
- `skills/start-feature/SKILL.md` — Phase 4 re-verify skip logic and Final Verification skip + marker write
- `hooks/scripts/quality-gate.js` — early-exit with marker file check

**Excluded:**
- Per-file lint hook (`hooks/scripts/lint-file.js`) — runs on individual file saves, not affected by this dedup
- Acceptance criteria verification stop hook — prompt-based, independent of quality gates
- `hooks/hooks.json` — no changes needed (stop hook stays, becomes a no-op when marker is fresh)
- Phase 4 iteration count changes — the fix-verify loop still runs max 3 iterations, just skips redundant quality gate re-runs within those iterations

## Expected Impact

| Metric | Before | After |
|--------|--------|-------|
| Typecheck runs per session | 6-8 | 2-3 |
| Lint runs per session | 2-3 | 1-2 |
| Wall-clock time saved | — | ~2-3 minutes |
| Token savings | — | ~5-10 API round-trips (LLM reading/interpreting redundant output) |
