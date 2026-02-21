# Parallelize Quality Gate Checks — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Run independent quality gate checks (typecheck, lint, type-sync) in parallel to reduce verification time from ~43s to ~25s per pass.

**Architecture:** Convert `execSync` calls to promisified `exec` in all check functions, wrap top-level orchestration in `async main()` using `Promise.allSettled`, and gate test execution on typecheck success.

**Tech Stack:** Node.js `child_process.exec`, `util.promisify`, `Promise.allSettled`

**Issue:** #42

---

### Task 1: Convert all check functions from sync to async

**Files:**
- Modify: `hooks/scripts/quality-gate.js:1-6` (imports)
- Modify: `hooks/scripts/quality-gate.js:28-49` (checkTypeScript)
- Modify: `hooks/scripts/quality-gate.js:53-85` (checkLint + runLintCommand)
- Modify: `hooks/scripts/quality-gate.js:89-140` (checkTypeSync + checkSupabaseTypes)
- Modify: `hooks/scripts/quality-gate.js:173-201` (checkTests)

**Step 1: Update imports**

Replace line 4:
```javascript
const { execSync } = require('child_process');
```

With:
```javascript
const { exec } = require('child_process');
const { promisify } = require('util');
const execAsync = promisify(exec);
```

**Step 2: Convert `checkTypeScript()` to async**

Replace lines 28-49 with:
```javascript
async function checkTypeScript() {
  const tsconfig = ['tsconfig.json', 'tsconfig.app.json', 'tsconfig.build.json'].find(f =>
    existsSync(f)
  );
  if (!tsconfig) return;
  if (!existsSync('node_modules/.bin/tsc')) return;

  try {
    await execAsync(`npx tsc --noEmit --project "${tsconfig}"`, { encoding: 'utf8' });
  } catch (e) {
    const output = execOutput(e);
    const errorLines = output.split('\n').filter(l => l.includes('error TS'));
    const count = errorLines.length;
    if (count > 0) {
      const shown = errorLines.slice(0, 20).join('\n  ');
      const more = count > 20 ? `\n  ... and ${count - 20} more` : '';
      failures.push(`[TSC] ${count} type error${count !== 1 ? 's' : ''}\n  ${shown}${more}`);
    } else if (output) {
      failures.push(`[TSC] TypeScript check failed\n  ${output.slice(0, 500)}`);
    }
  }
}
```

Note: `stdio` option is not needed for `exec` (it buffers stdout/stderr by default and returns them on the error object). The `execOutput(e)` helper works unchanged because promisified `exec` rejects with the same `.stdout`/`.stderr` properties.

**Step 3: Convert `runLintCommand()` and `checkLint()` to async**

Replace lines 53-85 with:
```javascript
async function checkLint() {
  // Try npm run lint first
  if (existsSync('package.json')) {
    try {
      const pkg = JSON.parse(readFileSync('package.json', 'utf8'));
      if (pkg.scripts?.lint) {
        await runLintCommand('npm run lint', 'Lint errors found');
        return;
      }
    } catch (e) {
      warnings.push(`[feature-flow] Failed to parse package.json: ${e.message?.slice(0, 100) || 'unknown'}. Falling back to direct linter detection.`);
    }
  }

  // Fallback: direct linter detection
  if (existsSync('node_modules/.bin/eslint') && hasEslintConfig()) {
    await runLintCommand('npx eslint .', 'ESLint errors');
    return;
  }

  if (existsSync('node_modules/.bin/biome') && hasBiomeConfig()) {
    await runLintCommand('npx biome check .', 'Biome errors');
  }
}

async function runLintCommand(command, label) {
  try {
    await execAsync(command, { encoding: 'utf8' });
  } catch (e) {
    const lines = execOutput(e).split('\n').slice(0, 20).join('\n  ');
    failures.push(`[LINT] ${label}\n  ${lines}`);
  }
}
```

**Step 4: Convert `checkTypeSync()` and `checkSupabaseTypes()` to async**

Replace lines 89-140 with:
```javascript
async function checkTypeSync() {
  if (!existsSync('.feature-flow.yml')) return;

  const yml = readFileSync('.feature-flow.yml', 'utf8');
  const stack = parseStack(yml);
  const typesPath = parseTypesPath(yml);

  if (stack.includes('supabase') && existsSync('supabase')) {
    await checkSupabaseTypes(typesPath);
  }

  if (stack.includes('prisma') && existsSync('prisma/schema.prisma')) {
    checkPrismaTypes();
  }

  checkDuplicateTypes(typesPath);
}

async function checkSupabaseTypes(typesPathOverride) {
  if (!existsSync('node_modules/.bin/supabase')) return;

  // Guard: is Supabase running?
  try {
    await execAsync('npx supabase status', { encoding: 'utf8' });
  } catch {
    warnings.push(
      '[feature-flow] Supabase not running locally — skipping type freshness check. Run "supabase start" to enable.'
    );
    return;
  }

  const typesFile = typesPathOverride || findTypesFile();
  if (!typesFile) {
    warnings.push('[feature-flow] No generated types file found — skipping Supabase type freshness check.');
    return;
  }

  try {
    const { stdout: fresh } = await execAsync('npx supabase gen types typescript --local', {
      encoding: 'utf8',
    });
    const existing = readFileSync(typesFile, 'utf8');
    if (fresh.trim() !== existing.trim()) {
      failures.push(
        `[TYPE-SYNC] Generated Supabase types are stale\n  Run: npx supabase gen types typescript --local > ${typesFile}`
      );
    }
  } catch (e) {
    warnings.push(`[feature-flow] Failed to generate Supabase types: ${(e.message || '').slice(0, 100)}`);
  }
}
```

Note: `checkSupabaseTypes` previously captured the full stdout from `execSync` return value. With `execAsync`, the resolved value is `{ stdout, stderr }`, so destructure `{ stdout: fresh }`.

`checkPrismaTypes()` and `checkDuplicateTypes()` stay synchronous — they use `statSync`/`readFileSync` only (no external processes).

**Step 5: Convert `checkTests()` to async**

Replace lines 173-201 with:
```javascript
async function checkTests() {
  const cmd = detectTestCommand();
  if (!cmd) return;

  try {
    await execAsync(cmd, {
      encoding: 'utf8',
      timeout: 60000,
      env: { ...process.env, CI: '1' },
    });
  } catch (e) {
    if (e.killed && e.signal === 'SIGTERM') {
      warnings.push('[feature-flow] Test suite timed out (60s) — skipping. Run tests manually.');
      return;
    }
    if (e.code === 'ENOENT' || e.status === 127) {
      warnings.push(`[feature-flow] Test command not found: "${cmd}". Ensure the tool is installed.`);
      return;
    }
    const output = execOutput(e);
    if (!output) {
      failures.push(`[TEST] Test suite failed: ${e.message?.slice(0, 200) || 'exit code ' + e.status}`);
    } else {
      const lines = output.split('\n').slice(0, 20).join('\n  ');
      failures.push(`[TEST] Test suite failed\n  ${lines}`);
    }
  }
}
```

**Step 6: Commit**

```bash
git add hooks/scripts/quality-gate.js
git commit -m "refactor: convert quality gate check functions from sync to async"
```

**Acceptance Criteria:**
- [ ] `execSync` does not appear anywhere in `hooks/scripts/quality-gate.js`
- [ ] `execAsync` is used in all places that previously used `execSync`
- [ ] `checkTypeScript`, `checkLint`, `runLintCommand`, `checkTypeSync`, `checkSupabaseTypes`, `checkTests` are all declared with `async function`
- [ ] `checkPrismaTypes` and `checkDuplicateTypes` remain synchronous (no external process calls)
- [ ] `checkSupabaseTypes` destructures `{ stdout: fresh }` from `execAsync` (not bare return value)
- [ ] `execOutput(e)` helper is unchanged
- [ ] `node -c hooks/scripts/quality-gate.js` exits with code 0 (no syntax errors)

---

### Task 2: Replace sequential top-level with async main() orchestration

**Files:**
- Modify: `hooks/scripts/quality-gate.js:8-24` (top-level orchestration)

**Step 1: Replace the sequential top-level calls with `async main()`**

Replace lines 8-24 (the `failures`/`warnings` declarations, sequential try/catch blocks, reporting, and `process.exit`) with:

```javascript
const failures = [];
const warnings = [];

async function main() {
  const results = await Promise.allSettled([
    checkTypeScript(),
    checkLint(),
    checkTypeSync(),
  ]);

  // Log unexpected crashes as warnings
  for (const r of results) {
    if (r.status === 'rejected') {
      warnings.push(`[feature-flow] Check failed unexpectedly: ${r.reason?.message?.slice(0, 100)}`);
    }
  }

  // Run tests only if typecheck passed (no type errors in failures)
  const hasTypeErrors = failures.some(f => f.startsWith('[TSC]'));
  if (!hasTypeErrors) {
    try {
      await checkTests();
    } catch (e) {
      warnings.push(`[feature-flow] Test check failed unexpectedly: ${e.message?.slice(0, 100)}`);
    }
  }

  if (failures.length > 0) {
    const report = failures.join('\n\n');
    const warn = warnings.length > 0 ? '\n\n' + warnings.join('\n') : '';
    console.log(`BLOCK: Code quality checks failed. Fix before ending session:\n\n${report}${warn}`);
  } else if (warnings.length > 0) {
    console.error(warnings.join('\n'));
  }
}

main().catch(e => {
  console.error(`[feature-flow] Quality gate crashed: ${e.message?.slice(0, 200)}`);
}).finally(() => {
  process.exit(0);
});
```

Key changes from original:
1. Three independent checks run in parallel via `Promise.allSettled`
2. `Promise.allSettled` replaces the per-function try/catch — rejected results become warnings
3. Tests are gated on typecheck: `failures.some(f => f.startsWith('[TSC]'))` checks if any TSC failures were recorded
4. `main().catch().finally()` ensures `process.exit(0)` always runs (hook protocol)

**Step 2: Verify syntax**

Run: `node -c hooks/scripts/quality-gate.js`
Expected: no output (clean syntax)

**Step 3: Commit**

```bash
git add hooks/scripts/quality-gate.js
git commit -m "perf: parallelize independent quality gate checks with Promise.allSettled

Typecheck, lint, and type-sync now run concurrently. Tests run sequentially
after, only if typecheck passed. Resolves #42."
```

**Acceptance Criteria:**
- [ ] `async function main()` exists and is called at module level
- [ ] `Promise.allSettled` is used with `checkTypeScript()`, `checkLint()`, `checkTypeSync()` as arguments
- [ ] `checkTests()` is NOT inside `Promise.allSettled` — it runs after, conditionally
- [ ] Tests are gated: only run when no `[TSC]` failures exist in the `failures` array
- [ ] `main().catch().finally()` pattern ensures `process.exit(0)` always runs
- [ ] Rejected `Promise.allSettled` results are pushed to `warnings` (not silently lost)
- [ ] `node -c hooks/scripts/quality-gate.js` exits with code 0 (no syntax errors)
- [ ] No sequential try/catch blocks remain at the top level (lines 11-14 of original are gone)

---

### Task 3: Add parallelization guidance to SKILL.md Phase 4

**Files:**
- Modify: `skills/start-feature/SKILL.md:714-727`

**Step 1: Add parallelization note to Phase 4 re-verify**

After line 717 (`After all fixes are applied, re-verify:`), before the `1. **Run tests:**` item, insert:

```markdown
**Parallelization:** When running quality checks inline, dispatch typecheck, lint, and type-sync as parallel Bash commands in a single message. These are independent checks. Only run tests after typecheck passes (tests depend on valid types).

```

The existing numbered list (1. Run tests, 2. Run verify-acceptance-criteria) remains unchanged below the new note.

**Step 2: Commit**

```bash
git add skills/start-feature/SKILL.md
git commit -m "docs: add parallelization guidance to quality gate verification steps"
```

**Acceptance Criteria:**
- [ ] `skills/start-feature/SKILL.md` contains the text "dispatch typecheck, lint, and type-sync as parallel Bash commands"
- [ ] The parallelization note appears between the "re-verify:" line and the "1. **Run tests:**" list item
- [ ] The existing Phase 4 content (test runner detection list, timeout, error handling, verify-acceptance-criteria) is unchanged
- [ ] Phase 5 (Report) section is unchanged
