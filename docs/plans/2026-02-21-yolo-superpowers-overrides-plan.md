# YOLO Superpowers Overrides — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add CRITICAL OVERRIDE blocks to start-feature/SKILL.md so YOLO mode suppresses all interactive prompts from superpowers skills

**Architecture:** Single-file edit — 6 changes to skills/start-feature/SKILL.md adding/strengthening YOLO override sections

**Tech Stack:** Markdown (SKILL.md content)

---

### Task 1: Strengthen brainstorming YOLO override

**Acceptance Criteria:**
- [ ] The YOLO behavior block in the "Brainstorming Interview Format Override" section contains "CRITICAL OVERRIDE" language
- [ ] The override text explicitly names all 4 brainstorming interactive instructions: "ask questions one at a time", "propose 2-3 approaches", "ask after each section", "Ready to set up for implementation?"
- [ ] The override includes 6 numbered steps for self-answering behavior
- [ ] The existing non-YOLO brainstorming format instructions are preserved unchanged

**Files:**
- Modify: `skills/start-feature/SKILL.md` — Replace the YOLO behavior block within the "Brainstorming Interview Format Override" section

**Steps:**
1. Find the current YOLO behavior block starting with `**YOLO behavior:** When YOLO mode is active (i.e., \`yolo: true\` is in the brainstorming args), do NOT present questions`
2. Replace it with the strengthened version from issue #30 Change 1
3. Verify the non-YOLO content above (format rules, question format) is unchanged

### Task 2: Add Writing Plans YOLO Override section

**Acceptance Criteria:**
- [ ] A new `### Writing Plans YOLO Override` section exists after the Brainstorming Interview Format Override section
- [ ] The section contains "CRITICAL OVERRIDE" language naming the "execution choice" prompt
- [ ] The section specifies 3 numbered steps: suppress prompt, announce, proceed to next step
- [ ] `grep -c "Writing Plans YOLO Override" skills/start-feature/SKILL.md` returns 1

**Files:**
- Modify: `skills/start-feature/SKILL.md` — Add new section after Brainstorming Interview Format Override

**Steps:**
1. Find the end of the Brainstorming Interview Format Override section (after the graduated YOLO checkpoint content)
2. Insert the new "Writing Plans YOLO Override" section from issue #30 Change 2

### Task 3: Add Using Git Worktrees YOLO Override section

**Acceptance Criteria:**
- [ ] A new `### Using Git Worktrees YOLO Override` section exists after the Writing Plans override
- [ ] The section contains "CRITICAL OVERRIDE" language naming both interactive prompts
- [ ] The section handles both worktree directory selection and baseline test failure
- [ ] `grep -c "Using Git Worktrees YOLO Override" skills/start-feature/SKILL.md` returns 1

**Files:**
- Modify: `skills/start-feature/SKILL.md` — Add new section after Writing Plans YOLO Override

**Steps:**
1. Insert the new section from issue #30 Change 3 after the Writing Plans YOLO Override section

### Task 4: Add Finishing a Development Branch YOLO Override section

**Acceptance Criteria:**
- [ ] A new `### Finishing a Development Branch YOLO Override` section exists after the Git Worktrees override
- [ ] The section contains "CRITICAL OVERRIDE" language naming the 4-option menu and base branch question
- [ ] The section specifies auto-selecting "Push and create a Pull Request" (Option 2)
- [ ] `grep -c "Finishing a Development Branch YOLO Override" skills/start-feature/SKILL.md` returns 1

**Files:**
- Modify: `skills/start-feature/SKILL.md` — Add new section after Using Git Worktrees YOLO Override

**Steps:**
1. Insert the new section from issue #30 Change 4 after the Git Worktrees YOLO Override section

### Task 5: Add Subagent-Driven Development YOLO Override section

**Acceptance Criteria:**
- [ ] A new `### Subagent-Driven Development YOLO Override` section exists after the Finishing override
- [ ] The section references the Finishing a Development Branch YOLO Override for sub-invocations
- [ ] The section specifies auto-answering implementer subagent questions
- [ ] `grep -c "Subagent-Driven Development YOLO Override" skills/start-feature/SKILL.md` returns 1

**Files:**
- Modify: `skills/start-feature/SKILL.md` — Add new section after Finishing a Development Branch YOLO Override

**Steps:**
1. Insert the new section from issue #30 Change 5 after the Finishing override section

### Task 6: Update YOLO Decision Log template

**Acceptance Criteria:**
- [ ] The YOLO Decision Log table in Step 5 Completion includes rows for brainstorming, writing-plans, using-git-worktrees, and finishing-a-dev-branch
- [ ] `grep -c "writing-plans" skills/start-feature/SKILL.md` is at least 2 (skill mapping + decision log)
- [ ] `grep -c "finishing-a-dev-branch" skills/start-feature/SKILL.md` is at least 2 (override section + decision log)

**Files:**
- Modify: `skills/start-feature/SKILL.md` — Update the YOLO Decision Log template in Step 5

**Steps:**
1. Find the YOLO Decision Log table template in the Step 5 Completion section
2. Add the 4 new rows from issue #30 Change 6
3. Update the "Total decisions auto-selected" description
