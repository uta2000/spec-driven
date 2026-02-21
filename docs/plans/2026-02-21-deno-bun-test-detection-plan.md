# Add Deno and Bun Test Runner Detection — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Deno and Bun to the test runner detection in `quality-gate.js` and the inline detection list in `SKILL.md`.

**Architecture:** Extend the existing `detectTestCommand()` cascade with two new entries (Deno, Bun) that use a config-file-then-runtime-check pattern. Update the SKILL.md documentation to match.

**Issue:** #40

---

### Task 1: Add `execSync` to imports

**Files:**
- Modify: `hooks/scripts/quality-gate.js:4`

**Step 1: Add `execSync` to the `child_process` import**

Change line 4 from:

```javascript
const { exec } = require('child_process');
```

to:

```javascript
const { exec, execSync } = require('child_process');
```

**Step 2: Verify the import parses correctly**

Run: `node -c hooks/scripts/quality-gate.js`
Expected: No output (syntax OK)

**Step 3: Commit**

```bash
git add hooks/scripts/quality-gate.js
git commit -m "chore: add execSync import for runtime detection (#40)"
```

**Acceptance Criteria:**

- [ ] `hooks/scripts/quality-gate.js` line 4 contains `exec, execSync` in the destructure
- [ ] `node -c hooks/scripts/quality-gate.js` exits with code 0

---

### Task 2: Add Deno detection to `detectTestCommand()`

**Files:**
- Modify: `hooks/scripts/quality-gate.js:260-262`

**Step 1: Add Deno detection block**

Insert the following **after** the Python check (after line 260, the closing `}` of the Python block) and **before** `return null` (line 262):

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

**Step 2: Verify the file parses correctly**

Run: `node -c hooks/scripts/quality-gate.js`
Expected: No output (syntax OK)

**Step 3: Verify detection logic with a quick smoke test**

Run: `node -e "const { existsSync } = require('fs'); const { execSync } = require('child_process'); console.log('existsSync:', typeof existsSync, 'execSync:', typeof execSync); console.log('OK')"`
Expected: `existsSync: function execSync: function` then `OK`

**Step 4: Commit**

```bash
git add hooks/scripts/quality-gate.js
git commit -m "feat: add Deno test runner detection (#40)"
```

**Acceptance Criteria:**

- [ ] `hooks/scripts/quality-gate.js` contains `existsSync('deno.json') || existsSync('deno.jsonc')` inside `detectTestCommand()`
- [ ] The Deno block returns `'deno test'` when `execSync('deno --version')` succeeds
- [ ] The Deno block pushes a warning containing `deno.json found but deno not installed` and returns `null` when `execSync` throws
- [ ] The Deno block appears after the Python check and before `return null`
- [ ] `node -c hooks/scripts/quality-gate.js` exits with code 0

---

### Task 3: Add Bun detection to `detectTestCommand()`

**Files:**
- Modify: `hooks/scripts/quality-gate.js` (after the Deno block, before `return null`)

**Step 1: Add Bun detection block**

Insert the following **after** the Deno block and **before** `return null`:

```javascript
  // Bun detection
  if (existsSync('bun.lockb') || existsSync('bunfig.toml')) {
    try {
      execSync('bun --version', { stdio: 'pipe', timeout: 5000 });
      return 'bun test';
    } catch {
      warnings.push('[feature-flow] bun.lockb found but bun not installed — skipping test check.');
      return null;
    }
  }
```

**Step 2: Verify the file parses correctly**

Run: `node -c hooks/scripts/quality-gate.js`
Expected: No output (syntax OK)

**Step 3: Commit**

```bash
git add hooks/scripts/quality-gate.js
git commit -m "feat: add Bun test runner detection (#40)"
```

**Acceptance Criteria:**

- [ ] `hooks/scripts/quality-gate.js` contains `existsSync('bun.lockb') || existsSync('bunfig.toml')` inside `detectTestCommand()`
- [ ] The Bun block returns `'bun test'` when `execSync('bun --version')` succeeds
- [ ] The Bun block pushes a warning containing `bun.lockb found but bun not installed` and returns `null` when `execSync` throws
- [ ] The Bun block appears after the Deno block and before `return null`
- [ ] `node -c hooks/scripts/quality-gate.js` exits with code 0

---

### Task 4: Update inline test detection list in SKILL.md

**Files:**
- Modify: `skills/start-feature/SKILL.md:725-726`

**Step 1: Add Deno and Bun entries to the inline test detection list**

Insert two new bullet points **after** the Python entry (line 725) and **before** the "If no test runner detected" line (line 726):

```markdown
   - `deno.json` / `deno.jsonc` → `deno test` (verify `deno` is installed first; if not, skip with warning)
   - `bun.lockb` / `bunfig.toml` → `bun test` (verify `bun` is installed first; if not, skip with warning)
```

**Step 2: Verify the entries match the bullet format of existing entries**

Read the modified section and confirm:
- Same indentation level (3 spaces + `-`)
- Same arrow notation (`→`)
- Parenthetical notes for runtime verification (consistent with how npm has `If node_modules doesn't exist, skip with warning`)

**Step 3: Commit**

```bash
git add skills/start-feature/SKILL.md
git commit -m "docs: add Deno and Bun to inline test detection list (#40)"
```

**Acceptance Criteria:**

- [ ] `skills/start-feature/SKILL.md` contains a line matching `deno.json` / `deno.jsonc` → `deno test`
- [ ] `skills/start-feature/SKILL.md` contains a line matching `bun.lockb` / `bunfig.toml` → `bun test`
- [ ] The new entries appear between the Python entry and the "If no test runner detected" entry
- [ ] The new entries use the same bullet format and indentation as existing entries
