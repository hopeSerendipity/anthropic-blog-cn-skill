# README Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add bilingual README documentation that explains the Anthropic blog translation collection, the notes files, and the local article workflow skill.

**Architecture:** Use two top-level README files with mirrored structure and cross-links. Keep the content repository-specific, with direct references to the existing article folders and the local skill path.

**Tech Stack:** Markdown, Git

---

### Task 1: Create the documentation files

**Files:**
- Create: `README.md`
- Create: `README.zh-CN.md`
- Modify: `docs/superpowers/specs/2026-04-14-readme-design.md`
- Modify: `docs/superpowers/plans/2026-04-14-readme-docs.md`

- [ ] **Step 1: Draft the English README**

Include:
- repo overview
- current article list
- folder layout
- skill overview
- contribution or extension flow

- [ ] **Step 2: Draft the Chinese README**

Mirror the English structure while adapting phrasing for Chinese readers.

- [ ] **Step 3: Add cross-links between language versions**

Add a visible switch near the top of both files.

- [ ] **Step 4: Check the paths and article names**

Confirm all listed article folders and the skill path exist.

- [ ] **Step 5: Review the rendered Markdown mentally**

Verify headings, lists, links, and code blocks are readable in plain Markdown.

### Task 2: Verify and commit

**Files:**
- Modify: `README.md`
- Modify: `README.zh-CN.md`

- [ ] **Step 1: Run repository checks**

Run:
- `git diff -- README.md README.zh-CN.md docs/superpowers/specs/2026-04-14-readme-design.md docs/superpowers/plans/2026-04-14-readme-docs.md`
- `git status --short`

Expected:
- only the intended documentation files show as added or modified

- [ ] **Step 2: Commit the documentation**

Run:
- `git add README.md README.zh-CN.md docs/superpowers/specs/2026-04-14-readme-design.md docs/superpowers/plans/2026-04-14-readme-docs.md`
- `git commit -m "docs: add bilingual repository readme"`
