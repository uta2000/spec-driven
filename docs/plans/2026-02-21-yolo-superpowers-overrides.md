# YOLO Superpowers Overrides — Design Document

**Date:** 2026-02-21
**Status:** Draft
**Issue:** #30

## Overview

YOLO mode pauses at 10 interactive points across 5 superpowers skills because the third-party skills have zero YOLO awareness. The fix adds "CRITICAL OVERRIDE" blocks to `start-feature/SKILL.md` that explicitly name and suppress conflicting interactive instructions from each superpowers skill. Two strategies: explicit pre-emption for brainstorming (complex self-answering), inline replacement for skills with deterministic correct answers.

## Root Cause

Override instructions in start-feature compete with skill-native instructions when both are in context. Weak language ("do not present", "skip this") non-deterministically loses. Fix uses "CRITICAL OVERRIDE" language that names the exact conflicting instruction.

## Changes to Existing Files

### `skills/start-feature/SKILL.md`

1. **Strengthen brainstorming YOLO override** — Replace weak 4-line YOLO block with 8-line "CRITICAL OVERRIDE" that names all 4 brainstorming interactive instructions to suppress
2. **Add Writing Plans YOLO Override section** — Suppress "execution choice" prompt, auto-select Subagent-Driven
3. **Add Using Git Worktrees YOLO Override section** — Auto-select `.worktrees/`, auto-proceed on test failure
4. **Add Finishing a Development Branch YOLO Override section** — Auto-select "Push and create PR", auto-confirm `main`
5. **Add Subagent-Driven Development YOLO Override section** — Apply finishing override to sub-invocation, auto-answer implementer questions
6. **Update YOLO Decision Log** — Add superpowers auto-decision rows to completion summary template

## Scope

**Included:**
- 6 edits to `skills/start-feature/SKILL.md`
- New "CRITICAL OVERRIDE" sections for 4 superpowers skills
- Strengthened brainstorming override
- Updated decision log template

**Excluded:**
- No modifications to superpowers skills (third-party)
- No changes to non-YOLO behavior
- No new files
