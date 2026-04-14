---
name: article-translation-notes
description: "Translate long-form articles or blog posts into Chinese, extract concise retrieval-friendly notes, and save both outputs as separate Markdown files. Use when Codex is given an article, essay, announcement, technical post, or transcript and the user wants: (1) a faithful Chinese translation, (2) a compact explanation of what the article is mainly about, (3) the author's key viewpoints distilled into short paragraph-style notes, or (4) both outputs organized into local files such as `translation.md` and `notes.md`."
---

# Article Translation Notes

## Overview

Turn a source article into two reusable artifacts: a faithful Chinese translation and a concise notes file optimized for later lookup. Preserve the article's structure in the translation, and compress the notes around the article's purpose, key arguments, and reusable ideas.

## Workflow

1. Read the full article before writing. Identify the title, the central claim, the intended audience, and whether the piece is mainly explanatory, persuasive, or procedural.
2. Translate into Chinese with fidelity first. Preserve the section hierarchy and meaning, prefer natural Chinese over word-for-word phrasing, and do not invent claims that are not present in the source.
3. Distill the notes for retrieval. Start with a short paragraph explaining what the article is mainly about, then capture the highest-value viewpoints as short sections with one paragraph each.
4. Save the outputs as separate Markdown files. Use a user-provided folder name when available; otherwise derive one from the article title. Default to `translation.md` for the full translation and `notes.md` for the distilled notes unless the user asks for a different layout.

## Translation Rules

- Preserve the source's section structure whenever practical.
- Translate for clarity and fluency in Chinese while keeping the original meaning intact.
- Keep product names, protocol names, and tool names accurate.
- Retain examples, caveats, comparisons, and workflow steps that matter to the article's logic.
- Keep important English terms in backticks on first mention when that helps later retrieval.

## Note-Taking Rules

- Optimize for later lookup rather than literary polish.
- Keep the notes lean and remove repetition or obvious marketing filler.
- Always answer "这篇文章主要讲什么" near the top.
- Focus on the article's reusable methods, heuristics, workflows, evaluation ideas, and other high-value viewpoints.
- Prefer short sections with one paragraph each. Use bullets only when the content is naturally list-shaped.
- When a point is abstract, explain it inside the same paragraph instead of splitting "观点" and "解释" into disconnected fragments.

## Output Shape

Use a simple two-file structure unless the user asks for something else.

`translation.md`
- Full Chinese translation with headings preserved.

`notes.md`
- A concise note file with a short "文章主旨" section first.
- A "核心观点" section after that.
- Each viewpoint written as a compact paragraph that combines the claim and why it matters.

## File and Folder Rules

- User-specified folder names take priority.
- If no folder name is given, derive one from the article title and sanitize it for the filesystem.
- If `translation.md` or `notes.md` already exist in the target folder, update them deliberately instead of creating accidental duplicates.

## Quality Checks

- Ensure the translation does not contradict or omit important parts of the source.
- Ensure the notes are substantially shorter than the translation.
- Ensure the notes clearly state the article's main topic and preserve the most useful viewpoints.
- Ensure each key viewpoint reads as a coherent paragraph rather than a heading plus sentence fragments.

## Example Requests

- "翻译这篇文章并做笔记。"
- "把这篇 blog 译成中文，然后提炼成便于复习的笔记。"
- "整理这篇 agent 文章，输出完整译文和精简总结，并分别写入两个文件。"
