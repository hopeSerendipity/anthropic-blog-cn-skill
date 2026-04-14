# Bilingual README Design

**Date:** 2026-04-14

## Goal

Add a clear bilingual README setup for this repository so new visitors can quickly understand:

- this repo contains Chinese translations of selected Anthropic blog posts
- each article also has a concise `notes.md` file for retrieval and review
- the repo includes a local Codex skill that supports the translation-plus-notes workflow

## Scope

This design covers:

- `README.md` as the English entry point
- `README.zh-CN.md` as the Chinese entry point
- cross-links between the two README files
- a concise explanation of the content layout and the local skill

## Structure

Both README files should include:

1. Repository overview
2. What each article folder contains
3. Current article list
4. Skill overview and workflow summary
5. Repo structure
6. How to extend the collection

## Constraints

- Keep the README practical and repository-specific
- Do not overstate automation beyond what exists in the repo
- Use the actual local skill path: `.agents/skills/article-translation-notes/SKILL.md`
- Reflect the current article set accurately
