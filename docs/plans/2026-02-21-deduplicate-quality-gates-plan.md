# Deduplicate Quality Gate Runs — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Eliminate redundant quality gate runs across lifecycle phases by adding commit-hash based skip logic at three dedup points.

**Architecture:** Add "skip if clean" instructions to two SKILL.md sections (Code Review Phase 4 re-verify and Final Verification) and add an early-exit marker check to `quality-gate.js`. All three dedup points share one mechanism: compare current HEAD hash / working tree state against the last successful verification.

**Tech Stack:** Node.js `child_process.execSync`, `fs.existsSync`/`readFileSync`, git CLI

**Design Doc:** `docs/plans/2026-02-21-deduplicate-quality-gates.md`

**Issue:** #36

---

### Task 1: Add early-exit marker check to quality-gate.js

**Files:**
- Modify: `hooks/scripts/quality-gate.js:7` (add `writeFileSync` to imports)
- Modify: `hooks/scripts/quality-gate.js:13` (add early-exit at top of `main()`)

**Step 1: Add `writeFileSync` to the fs import**

Replace line 7:
```javascript
const { existsSync, readFileSync, readdirSync, statSync } = require('fs');
```

With:
```javascript
const { existsSync, readFileSync, writeFileSync, readdirSync, statSync } = require('fs');
```

**Step 2: Add early-exit logic at the top of `main()`**

Insert after line 13 (`async function main() {`), before the `const checks = [` line:

```javascript
  // Skip if lifecycle already verified at this commit with clean working tree
  try {
    const markerPath = path.join(
      execSync('git rev-parse --git-dir', { encoding: 'utf8' }).trim(),
      'feature-flow-verified'
    );
    if (existsSync(markerPath)) {
      const savedHash = readFileSync(markerPath, 'utf8').trim();
      const currentHash = execSync('git rev-parse HEAD', { encoding: 'utf8' }).trim();
      const dirty = execSync('git diff HEAD --name-only', { encoding: 'utf8' }).trim();
      if (savedHash === currentHash && !dirty) {
        return; // All checks passed at this commit, working tree clean
      }
    }
  } catch {
    // Fall through to run checks — fail-open on any git/fs error
  }

```

Note: Uses `git rev-parse --git-dir` instead of hardcoded `.git/` to handle worktrees (where `.git` is a file pointing to the main repo's `.git/worktrees/` directory). `execSync` is already imported (line 4). The `path` module is already imported (line 8).

**Step 3: Add marker-write function**

Insert after the `main().catch().finally()` block (after line 49), before the `// --- Check 1: TypeScript ---` comment:

```javascript

// --- Marker file ---

function writeVerificationMarker() {
  try {
    const markerPath = path.join(
      execSync('git rev-parse --git-dir', { encoding: 'utf8' }).trim(),
      'feature-flow-verified'
    );
    const hash = execSync('git rev-parse HEAD', { encoding: 'utf8' }).trim();
    writeFileSync(markerPath, hash + '\n');
  } catch {
    // Non-critical — marker write failure doesn't affect quality gate results
  }
}
```

**Step 4: Call `writeVerificationMarker()` after all checks pass**

In the `main()` function, after the reporting block (the `if (failures.length > 0)` / `else if` / block ending around line 42), add the marker write when all checks pass. Replace the end of the reporting block:

The current code ends with:
```javascript
  } else if (warnings.length > 0) {
    console.error(warnings.join('\n'));
  }
}
```

Replace with:
```javascript
  } else if (warnings.length > 0) {
    console.error(warnings.join('\n'));
  }

  // Write verification marker when all checks pass (warnings are OK, failures are not)
  if (failures.length === 0) {
    writeVerificationMarker();
  }
}
```

**Step 5: Verify syntax**

Run: `node -c hooks/scripts/quality-gate.js`
Expected: no output (clean syntax)

**Step 6: Commit**

```bash
git add hooks/scripts/quality-gate.js
git commit -m "perf: add commit-hash marker for quality gate deduplication

Stop hook now writes .git/feature-flow-verified after passing checks and
skips re-running when HEAD matches and working tree is clean. Resolves #36."
```

**Acceptance Criteria:**
- [ ] `writeFileSync` is imported from `fs` in the destructured import on line 7
- [ ] `main()` has an early-exit block that reads `.git/feature-flow-verified` (via `git rev-parse --git-dir`), compares saved hash with `git rev-parse HEAD`, and checks `git diff HEAD --name-only` for dirty files
- [ ] Early-exit returns (skips all checks) only when saved hash matches current HEAD AND working tree is clean
- [ ] Early-exit is wrapped in a try/catch that falls through to run checks on any error (fail-open)
- [ ] `writeVerificationMarker()` function exists and writes the HEAD hash to `.git/feature-flow-verified`
- [ ] `writeVerificationMarker()` is called when `failures.length === 0` (after all checks complete)
- [ ] `writeVerificationMarker()` is NOT called when there are failures
- [ ] `writeVerificationMarker()` uses `git rev-parse --git-dir` (not hardcoded `.git/`) to support worktrees
- [ ] `node -c hooks/scripts/quality-gate.js` exits with code 0 (no syntax errors)

---

### Task 2: Add skip-if-clean logic to Code Review Phase 4 re-verify

**Files:**
- Modify: `skills/start-feature/SKILL.md:714-737` (Phase 4: Re-verify section)

**Step 1: Add skip-if-clean instruction to the re-verify section**

After line 718 (the `**Parallelization:**` paragraph) and before line 720 (`1. **Run tests:**`), insert:

```markdown

**Skip if clean:** Before running quality checks in each iteration, check `git diff --stat` against the last verified state. If no source files changed since the last successful quality gate pass within this pipeline, skip the quality gate re-run and announce: "Quality gates passed at [commit] — no changes since last check. Skipping re-verify." Only run `verify-acceptance-criteria` (step 2 below).

```

The existing numbered list (1. Run tests, 2. Run verify-acceptance-criteria) remains unchanged below this new paragraph.

**Step 2: Commit**

```bash
git add skills/start-feature/SKILL.md
git commit -m "perf: add skip-if-clean check to code review Phase 4 re-verify

Skips redundant quality gate re-runs between fix-verify iterations when
no source files changed since the last successful pass."
```

**Acceptance Criteria:**
- [ ] `skills/start-feature/SKILL.md` contains the text "Skip if clean:" between the "Parallelization:" paragraph and the "1. **Run tests:**" list item
- [ ] The skip instruction references `git diff --stat` for checking changes
- [ ] The skip instruction says to still run `verify-acceptance-criteria` even when skipping quality gates
- [ ] The existing Phase 4 content (test runner detection list, timeout, error handling, verify-acceptance-criteria, iteration loop) is unchanged
- [ ] Phase 5 (Report) section is unchanged

---

### Task 3: Add skip and marker-write logic to Final Verification step

**Files:**
- Modify: `skills/start-feature/SKILL.md:344` (Final verification row in Skill Mapping table)

**Step 1: Add an inline Final Verification section**

The current Final Verification is a single row in the Skill Mapping table (line 344). It needs expanded instructions. After the Generate CHANGELOG Entry inline section (which ends before the `### Step 4: Handle Interruptions` section), insert a new inline section:

Find the line:
```markdown
### Step 4: Handle Interruptions
```

Insert before it:

```markdown
### Final Verification Step (inline — no separate skill)

This step runs after CHANGELOG generation and before commit and PR. It verifies acceptance criteria and runs quality gates — but skips redundant quality gate runs when the code review pipeline already passed them.

**Process:**

1. **Check for redundant quality gates:** Before running `verification-before-completion` (which runs typecheck, lint, build), check if the Code Review Pipeline's Phase 4 already passed these checks in this lifecycle. If it did, check `git status --porcelain`:
   - If output is empty (no modifications since Phase 4): Skip `verification-before-completion`. Announce: "Quality gates already passed in code review Phase 4 — no changes since. Skipping redundant checks."
   - If output is non-empty: Run `verification-before-completion` normally (files changed since Phase 4).

2. **Always run `verify-acceptance-criteria`:** This checks plan-specific criteria and must always run regardless of quality gate skip.

3. **Write verification marker:** After all checks pass (both acceptance criteria and any quality gates that ran), write the HEAD commit hash to the git directory for the stop hook to read:
   ```bash
   git rev-parse HEAD > "$(git rev-parse --git-dir)/feature-flow-verified"
   ```
   This prevents the stop hook from re-running the same quality gates when the session ends.

```

**Step 2: Update the Skill Mapping table row**

Replace the Final Verification row (line 344):

```markdown
| Final verification | `feature-flow:verify-acceptance-criteria` + `superpowers:verification-before-completion` | All criteria PASS + lint/typecheck/build pass |
```

With:

```markdown
| Final verification | No skill — inline step (see below) | All criteria PASS + quality gates pass (or skipped if Phase 4 already passed) |
```

**Step 3: Commit**

```bash
git add skills/start-feature/SKILL.md
git commit -m "perf: add quality gate skip and marker write to Final Verification

Final Verification now skips redundant quality gates when code review
Phase 4 already passed them and no files changed since. Writes a
verification marker for the stop hook to read."
```

**Acceptance Criteria:**
- [ ] A `### Final Verification Step (inline — no separate skill)` section exists in SKILL.md
- [ ] The section appears after the Generate CHANGELOG Entry section and before `### Step 4: Handle Interruptions`
- [ ] The section instructs to check `git status --porcelain` before running `verification-before-completion`
- [ ] The section instructs to skip `verification-before-completion` when Phase 4 already passed and working tree is clean
- [ ] The section instructs to always run `verify-acceptance-criteria` regardless of skip
- [ ] The section includes the marker write command: `git rev-parse HEAD > "$(git rev-parse --git-dir)/feature-flow-verified"`
- [ ] The Skill Mapping table row for Final Verification is updated to "No skill — inline step (see below)"
- [ ] The updated table row mentions "skipped if Phase 4 already passed"
- [ ] `node -c hooks/scripts/quality-gate.js` still exits with code 0 (syntax unaffected)
