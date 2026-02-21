# Add Deno and Bun Test Runner Detection — Design Document

**Date:** 2026-02-21
**Status:** Draft
**Issue:** #40

## Overview

Add Deno and Bun to the test runner detection logic in `hooks/scripts/quality-gate.js` and the inline test detection list in `skills/start-feature/SKILL.md`. Both runtimes are currently missing, causing noisy errors when the LLM probes for runtimes outside the prescribed detection logic. The fix follows the existing pattern: detect config file → verify runtime installed → return command or warn+skip.

## User Flow

### Step 1 — Quality Gate Runs on Commit

The `quality-gate.js` hook fires during a stop event. It calls `detectTestCommand()` to find the project's test runner.

### Step 2 — Deno/Bun Project Detected

If the project has `deno.json`/`deno.jsonc` or `bun.lockb`/`bunfig.toml`, the detection function now recognizes these as Deno or Bun projects.

### Step 3 — Runtime Verified Before Command Returned

Before returning `deno test` or `bun test`, the function verifies the runtime is actually installed using `execSync('<runtime> --version')`. If not installed, it pushes a warning and returns `null` (skip), preventing exit code 1/127 errors.

## Pipeline / Architecture

### Detection Order in `detectTestCommand()`

The new runtimes are added **after** the existing Python check and **before** `return null`, preserving the existing cascade:

1. `package.json` → `npm test`
2. `Cargo.toml` → `cargo test`
3. `go.mod` → `go test ./...`
4. `mix.exs` → `mix test`
5. `pyproject.toml` / `pytest.ini` / `setup.cfg` / `tox.ini` → `python -m pytest`
6. **`deno.json` / `deno.jsonc` → `deno test`** (new)
7. **`bun.lockb` / `bunfig.toml` → `bun test`** (new)
8. `return null`

### Runtime Verification Pattern

Both new entries use `execSync` with a try/catch to verify the runtime binary exists before returning the command. This is different from existing entries (which assume the runtime is installed if the config file exists) but necessary because Deno and Bun config files can exist in monorepos or shared environments where the runtime isn't available.

```javascript
// Deno detection
if (existsSync('deno.json') || existsSync('deno.jsonc')) {
  try {
    execSync('deno --version', { stdio: 'pipe', timeout: 5000 });
    return 'deno test';
  } catch {
    warnings.push('[feature-flow] deno.json found but deno not installed — skipping test check.');
    return null;
  }
}
```

### Import Addition

`execSync` must be added to the `child_process` import on line 4. Currently only `exec` is imported:

```javascript
// Before:
const { exec } = require('child_process');
// After:
const { exec, execSync } = require('child_process');
```

## Scope

### Included
- Deno detection (`deno.json`, `deno.jsonc`) in `detectTestCommand()`
- Bun detection (`bun.lockb`, `bunfig.toml`) in `detectTestCommand()`
- Runtime verification (binary existence check) for both
- Warning messages when config exists but runtime is missing
- `execSync` import addition
- SKILL.md inline test detection list update (lines 720-725)

### Excluded
- Bun vs npm priority when both `bun.lockb` and `package.json` exist (separate concern)
- Deno/Bun lint detection (out of scope for this issue)
- Changes to `checkTests()` execution logic (only `detectTestCommand()` changes)

## Files to Modify

- `hooks/scripts/quality-gate.js` — `detectTestCommand()` function (line 238-262) + import (line 4)
- `skills/start-feature/SKILL.md` — Inline test detection list (lines 720-725)
