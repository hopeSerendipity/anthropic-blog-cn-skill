# anthropic-blog-cn-skill

[中文版本](README.zh-CN.md)

This repository collects Chinese translations and concise notes for selected Anthropic blog posts. It also includes a local Codex skill for turning long-form articles into two reusable outputs: a faithful Chinese translation and a compact notes file optimized for later lookup.

## What is in this repo

Each article lives in its own folder and usually contains:

- `translation.md`: the full Chinese translation
- `notes.md`: short retrieval-friendly notes distilled from the article

Current article set:

- [Building agents with the Claude Agent SDK](Building%20agents%20with%20the%20Claude%20Agent%20SDK/)
- [Effective context engineering for AI agents](Effective%20context%20engineering%20for%20AI%20agents/)
- [Introducing Contextual Retrieval](Introducing%20Contextual%20Retrieval/)
- [Introducing advanced tool use on the Claude Developer Platform](Introducing%20advanced%20tool%20use%20on%20the%20Claude%20Developer%20Platform/)
- [The think tool Enabling Claude to stop and think in complex tool use situations](The%20think%20tool%20Enabling%20Claude%20to%20stop%20and%20think%20in%20complex%20tool%20use%20situations/)
- [Writing effective tools for agents](Writing%20effective%20tools%20for%20agents/)

## Included skill

The repository also ships with a local skill at `.agents/skills/article-translation-notes/SKILL.md`.

That skill is designed for a simple workflow:

1. Read the source article in full.
2. Produce a faithful Chinese translation.
3. Distill the article into concise, reusable notes.
4. Save the outputs as `translation.md` and `notes.md` in the target folder.

This makes the repo useful both as a reading archive and as a reusable workflow template for future article curation.

## Repository layout

```text
.
├── .agents/
│   └── skills/
│       └── article-translation-notes/
│           └── SKILL.md
├── Building agents with the Claude Agent SDK/
│   ├── notes.md
│   └── translation.md
├── Effective context engineering for AI agents/
│   ├── notes.md
│   └── translation.md
├── ...
└── README.zh-CN.md
```

## How to extend the collection

To add a new Anthropic blog post or another long-form article:

1. Create a folder named after the article title.
2. Generate or write `translation.md` for the full Chinese translation.
3. Generate or write `notes.md` for the concise notes.
4. Update the article list in both README files if you want the new entry to appear in the index.

## Why this repo exists

This repo is meant to be a lightweight knowledge base for reading, review, and later retrieval. The combination of full translation, compact notes, and a reusable skill keeps the workflow easy to repeat as more articles are added.
