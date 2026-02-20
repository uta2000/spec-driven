# Structural Anti-Pattern Awareness — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Embed structural anti-pattern awareness into three existing lifecycle phases so that coupling, cohesion, and dependency direction are assessed at design time, pattern study, and self-review — not just as a post-hoc code review.

**Architecture:** Four markdown file modifications. No code, no tests, no data model. Each task adds a specific section to an existing file, following the file's established formatting conventions.

**Tech Stack:** Markdown (skill definitions, reference docs)

**Issue:** #18

---

### Task 1: Add Category 14 to Design Verification Checklist

**Files:**
- Modify: `skills/design-verification/references/checklist.md` (append after line 217, the end of category 13)

**Step 1: Add the new category section**

Append the following after the last line of `## 13. Route & Layout Chain`:

```markdown

## 14. Structural Anti-Patterns

For every new file, component, module, or class proposed in the design:

- [ ] **God objects/files:** No single file or component takes on more than 2-3 responsibilities. If the design concentrates logic in one place, flag it.
- [ ] **Tight coupling:** New modules don't depend on implementation details of other modules. Check for designs that wire directly to database columns, internal state, or private methods of other components.
- [ ] **Circular dependencies:** The proposed import/dependency graph has no cycles. If module A will import from B and B will import from A (directly or transitively), flag it.
- [ ] **Dependency direction:** Dependencies flow from high-level (UI, orchestration) to low-level (utilities, data access), not the reverse. Shared types/interfaces live in a common layer, not in a consumer.
- [ ] **Responsibility distribution:** Logic is distributed across appropriate layers (data access, business logic, presentation) rather than concentrated in a single file or function.

**Severity mapping:**
- God objects with 4+ responsibilities → **FAIL** (must restructure before implementation)
- Circular dependencies → **FAIL** (must redesign module boundaries)
- Tight coupling to implementation details → **WARNING** (recommend abstraction, not blocking)
- Minor responsibility distribution concerns → **WARNING**

**Where to look:**
- The design's "New Components" and "Pipeline / Architecture" sections
- Proposed file paths and their import relationships
- Any single file the design touches for more than 2 distinct concerns

**Common findings:**
- Design creates a single "manager" or "handler" file that fetches, transforms, validates, caches, and renders — a god object
- New utility imports from a UI component instead of a shared layer — inverted dependency
- Feature A imports a helper from Feature B, and Feature B imports a type from Feature A — circular dependency
- All business logic lives in an API route handler instead of an extracted service layer — mixed responsibilities
```

**Step 2: Verify the edit**

Run: `grep -c "## 14. Structural Anti-Patterns" skills/design-verification/references/checklist.md`
Expected: `1`

Run: `grep -c "God objects/files" skills/design-verification/references/checklist.md`
Expected: `1`

**Step 3: Commit**

```bash
git add skills/design-verification/references/checklist.md
git commit -m "feat: add structural anti-pattern category 14 to design verification checklist"
```

**Acceptance Criteria:**
- [ ] `skills/design-verification/references/checklist.md` contains a `## 14. Structural Anti-Patterns` section with 5 checks (god objects, tight coupling, circular deps, dependency direction, responsibility distribution)
- [ ] Category 14 severity mapping: god objects with 4+ responsibilities → FAIL, circular deps → FAIL, tight coupling → WARNING

---

### Task 2: Reference Category 18 in Design Verification SKILL.md

**Files:**
- Modify: `skills/design-verification/SKILL.md` (Step 4 base category list around line 69-83, and verification depth table around lines 174-179)

**Step 1: Add category 18 to the base checklist**

After line 83 (`13. **Route & Layout Chain** — New pages inherit auth, layout, providers correctly`), add:

```markdown
14. **Structural Anti-Patterns** — God objects, tight coupling, circular dependencies, dependency direction
```

**Important:** This renumbers the existing context-dependent categories. The current 14-17 become 15-18. Update the numbering:

- Current `14. **Stack-Specific Checks**` → `15. **Stack-Specific Checks**`
- Current `15. **Platform-Specific Checks**` → `16. **Platform-Specific Checks**`
- Current `16. **Project Gotchas**` → `17. **Project Gotchas**`
- Current `17. **Documentation Compliance (Context7)**` → `18. **Documentation Compliance (Context7)**`

**Step 2: Update the verification depth table**

Replace the current depth table (lines 174-179) with:

```markdown
| Design Scope | Depth |
|-------------|-------|
| New page with new data model | Full checklist (all 14 base categories + stack/platform/gotchas + doc compliance) |
| New API route, existing data model | Categories 1-3, 5, 7-8, 10-12, 14, 18 + stack/platform/gotchas |
| UI-only change, no schema changes | Categories 4-6, 9-10, 12-14 + platform/gotchas |
| Configuration or env change | Categories 7, 10-12, 14 + stack/gotchas |
```

Note: Category 14 (Structural Anti-Patterns) is included in ALL scope levels.

**Step 3: Update the Additional Resources reference**

The reference at line 193 says "Base verification checklist with 13 categories". Update to say "14 categories":

```markdown
- **`references/checklist.md`** — Base verification checklist with 14 categories, specific checks, and examples of common findings
```

**Step 4: Verify the edits**

Run: `grep "Structural Anti-Patterns" skills/design-verification/SKILL.md`
Expected: line with `14. **Structural Anti-Patterns**`

Run: `grep "all 14 base categories" skills/design-verification/SKILL.md`
Expected: `1` match

**Step 5: Commit**

```bash
git add skills/design-verification/SKILL.md
git commit -m "feat: reference structural anti-pattern category in design verification skill"
```

**Acceptance Criteria:**
- [ ] `skills/design-verification/SKILL.md` Step 4 references category 14 (Structural Anti-Patterns) in the base checklist
- [ ] Existing context-dependent categories renumbered from 14-17 to 15-18
- [ ] Verification depth table includes category 14 for all scope levels
- [ ] Additional Resources reference updated from "13 categories" to "14 categories"

---

### Task 3: Amend Study Existing Patterns + Self-Review in start-feature

**Files:**
- Modify: `skills/start-feature/SKILL.md` (Study Existing Patterns step lines ~315-368, Self-Review step lines ~370-408)

**Step 1: Add anti-pattern flagging sub-step to Study Existing Patterns**

After the current step 5 (line 348, "Generate 'How to Code This' notes") and before step 6 (line 363, "Pass these patterns AND..."), insert a new step. Renumber accordingly — current step 5 stays as 5, new step becomes 6, current step 6 becomes 7.

Insert after the "How to Code This" code block (after line 361):

```markdown

6. **Flag anti-patterns in existing code (do not replicate):**
   For each file read, assess whether it exhibits structural issues:
   - File exceeds 300 lines → note as a god file; do not replicate the monolithic structure
   - File mixes concerns (e.g., data fetching + rendering + business logic in one component) → note which concern to extract in new code
   - File has imports that create circular dependencies → note and avoid in new code
   - File duplicates logic found elsewhere → note the canonical location to import from instead

   If anti-patterns are found, add an "Anti-Patterns to Avoid" section to the output:

```
### Anti-Patterns Found (do NOT replicate)
- `src/components/SearchPage.tsx` (412 lines) — god component mixing fetch, transform, and render. New code should separate these into a hook (fetch/transform) and a component (render).
- `src/lib/utils.ts` imports from `src/components/Badge.tsx` — inverted dependency. New utilities must not import from components.
```

   Pass these warnings alongside the positive patterns to the implementation step as mandatory context. New code must follow the good patterns AND avoid the flagged anti-patterns.
```

Update the existing step 6 to become step 7:
```markdown
7. Pass these patterns, the "How to Code This" notes, AND any anti-pattern warnings to the implementation step as mandatory context. **New code MUST follow these patterns unless there is a documented reason to deviate.**
```

**Step 2: Add consistency exception to quality rules**

After the existing quality rule at line 368 (`- If existing patterns conflict with coding-standards.md, note the conflict and follow the existing codebase pattern (consistency > purity)`), add:

```markdown
- If existing patterns conflict with structural quality (god files, tight coupling), document the conflict. New code follows the better pattern, not the existing anti-pattern. Note: this is the ONE exception to the "consistency > purity" rule — structural anti-patterns should not be replicated even for consistency.
```

**Step 3: Add 4 structural items to the Self-Review checklist**

After line 389 (`- [ ] **Imports organized:** External, internal, relative, types — in that order.`), add:

```markdown
- [ ] **No god files:** No new file exceeds 300 lines or handles more than 2-3 responsibilities.
- [ ] **No circular dependencies:** New imports don't create cycles. If file A imports from file B, file B does not import (directly or transitively) from file A.
- [ ] **Dependency direction:** New code depends on abstractions (interfaces, types, shared utilities), not on implementation details of other features.
- [ ] **Duplication across files:** No logic block is copy-pasted from another file. If similar logic exists, import from the canonical source or extract a shared utility.
```

**Step 4: Verify the edits**

Run: `grep "Anti-Patterns Found" skills/start-feature/SKILL.md`
Expected: at least 1 match

Run: `grep "No god files" skills/start-feature/SKILL.md`
Expected: 1 match

Run: `grep "No circular dependencies" skills/start-feature/SKILL.md`
Expected: 1 match

Run: `grep "consistency > purity" skills/start-feature/SKILL.md`
Expected: 2 matches (original rule + the exception note)

**Step 5: Commit**

```bash
git add skills/start-feature/SKILL.md
git commit -m "feat: add structural anti-pattern awareness to study patterns and self-review steps"
```

**Acceptance Criteria:**
- [ ] `skills/start-feature/SKILL.md` Study Existing Patterns step includes anti-pattern flagging sub-step with "Anti-Patterns Found (do NOT replicate)" output section
- [ ] `skills/start-feature/SKILL.md` Study Existing Patterns quality rules include the consistency exception for structural anti-patterns
- [ ] `skills/start-feature/SKILL.md` Self-Review checklist has 14 items (10 existing + 4 structural: no god files, no circular deps, dependency direction, duplication across files)

---

### Task 4: Add Structural Quality Section to Coding Standards

**Files:**
- Modify: `references/coding-standards.md` (insert after "Separation of Concerns" section, before "Naming Conventions")

**Step 1: Add the new section**

After line 100 (`- **Configuration separate from code:** API keys, URLs, thresholds go in environment variables or config objects, not inline.`) and before line 102 (`## Naming Conventions`), insert:

```markdown

## Structural Quality

- **No god objects:** A file, class, or component should have 2-3 responsibilities at most. If you need "and" more than twice to describe what it does, split it.
- **Dependency direction matters:** High-level modules (pages, orchestrators) depend on low-level modules (utilities, data access), never the reverse. Shared types live in a common layer.
- **No circular dependencies:** If module A imports from B, B must not import from A. Use dependency inversion (shared interface in a third module) to break cycles.
- **Explicit boundaries:** Features should be self-contained directories. Cross-feature imports go through public APIs (barrel exports), not deep file paths.
- **Colocation over centralization:** Put code where it's used. A utility used by one feature lives in that feature's directory, not in a global utils/ folder. Promote to shared only when a second consumer appears.
```

**Step 2: Verify the edit**

Run: `grep "## Structural Quality" references/coding-standards.md`
Expected: 1 match

Run: `grep "No god objects" references/coding-standards.md`
Expected: 1 match

Run: `grep "Colocation over centralization" references/coding-standards.md`
Expected: 1 match

**Step 3: Commit**

```bash
git add references/coding-standards.md
git commit -m "feat: add structural quality section to coding standards"
```

**Acceptance Criteria:**
- [ ] `references/coding-standards.md` contains a `## Structural Quality` section with 5 principles (no god objects, dependency direction, no circular deps, explicit boundaries, colocation)
- [ ] Section is positioned after "Separation of Concerns" and before "Naming Conventions"
