# anthropic-blog-cn-skill

[English Version](README.md)

这个仓库收录了一些 Anthropic 博文的中文翻译与精炼笔记，同时也包含一个本地 Codex skill，用来把长文章整理成两份可复用产物：完整中文译文和便于后续检索的 notes。

## 仓库里有什么

每篇文章单独放在一个目录里，通常包含：

- `translation.md`：完整中文翻译
- `notes.md`：适合复习与检索的精炼笔记

当前已收录文章：

- [Building agents with the Claude Agent SDK](Building%20agents%20with%20the%20Claude%20Agent%20SDK/)
- [Effective context engineering for AI agents](Effective%20context%20engineering%20for%20AI%20agents/)
- [Introducing Contextual Retrieval](Introducing%20Contextual%20Retrieval/)
- [Introducing advanced tool use on the Claude Developer Platform](Introducing%20advanced%20tool%20use%20on%20the%20Claude%20Developer%20Platform/)
- [The think tool Enabling Claude to stop and think in complex tool use situations](The%20think%20tool%20Enabling%20Claude%20to%20stop%20and%20think%20in%20complex%20tool%20use%20situations/)
- [Writing effective tools for agents](Writing%20effective%20tools%20for%20agents/)

## 配套 skill

仓库内置了一个本地 skill，路径是 `.agents/skills/article-translation-notes/SKILL.md`。

这个 skill 对应的流程很直接：

1. 完整阅读源文章
2. 生成忠实的中文翻译
3. 提炼出简洁、可复用的笔记
4. 将结果分别保存为目标目录下的 `translation.md` 和 `notes.md`

所以这个仓库不仅是 Anthropic blog 的中文整理仓库，也可以视为一个可复用的文章翻译与做笔记工作流样板。

## 仓库结构

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
└── README.md
```

## 如何继续扩展

如果后续想继续加入新的 Anthropic 博文或其他长文，可以按下面的方式扩展：

1. 用文章标题创建一个新目录
2. 在目录中加入完整译文 `translation.md`
3. 在目录中加入精炼笔记 `notes.md`
4. 如果希望索引里显示新文章，再同步更新中英文 README 中的文章列表

## 这个仓库的用途

这个仓库的目标是做一个轻量、可持续扩展的知识库。完整译文负责保留细节，精炼 notes 负责提升复习与检索效率，而 skill 则把这套流程沉淀成可以重复使用的方法。
