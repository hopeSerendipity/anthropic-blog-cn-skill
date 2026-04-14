# Effective harnesses for long-running agents - 中文翻译

发布时间：`2025-11-26`

Agent 在跨越多个上下文窗口持续工作时，仍然会面临挑战。为了给长时间运行的 agent 构建更有效的 harness，我们从人类工程师的工作方式中寻找灵感。

随着 AI agent 越来越强，开发者也越来越希望它们承担那些需要持续数小时、甚至数天的复杂任务。然而，如何让 agent 在跨越多个上下文窗口的情况下依然持续、稳定地推进工作，仍然是一个尚未完全解决的问题。

长时间运行 agent 的核心挑战在于：它们必须以离散 session 的方式工作，而每个新 session 一开始都对之前发生过什么毫无记忆。你可以把它想象成一个软件项目由轮班工程师完成，而每个接班的工程师在上岗时都完全不知道前一班做了什么。由于上下文窗口有限，而多数复杂项目又不可能在单个窗口中完成，因此 agent 需要一种方式来跨越不同编码 session 之间的断层。

我们开发了一套双重方案，使 Claude Agent SDK 能在多个上下文窗口之间有效工作：一个 `initializer agent` 在第一次运行时负责搭建环境；一个 `coding agent` 则在之后的每个 session 中专注于做增量进展，并为下一个 session 留下清晰的工作痕迹。你可以在配套的 quickstart 中找到代码示例。

## 长时间运行 agent 的问题

Claude Agent SDK 是一个强大且通用的 agent harness，既擅长编程，也适合处理其他需要模型通过工具收集上下文、制定计划并执行的任务。它具备一些上下文管理能力，例如 `compaction`，使 agent 可以在不耗尽上下文窗口的情况下继续处理任务。从理论上说，凭借这套机制，agent 应该可以在任意长的时间里持续做有用的工作。

然而，仅靠 compaction 还不够。实际测试中，即使是像 Opus 4.5 这样的前沿编程模型，运行在 Claude Agent SDK 的循环里、跨越多个上下文窗口工作时，如果只给它一个高层 prompt，比如“做一个 claude.ai 的克隆版”，也依然无法稳定构建出生产质量的 Web 应用。

Claude 的失败主要呈现出两种模式。第一种是，agent 往往试图一次性做太多事情，本质上是在试图“一步到位”地做完整个应用。这样很容易导致模型在实现过程中途耗尽上下文，让下一个 session 接手时面对的是一个做到一半、而且没有文档说明的功能。接下来的 agent 就不得不猜测之前发生了什么，并花大量时间把基础应用重新修到可运行状态。即使有 compaction，这种问题也依然会发生，因为 compaction 并不总能把极其清晰的指令传递给下一位 agent。

第二种失败模式通常发生在项目后期。当系统已经做出一些功能之后，后续某个 agent 进入项目，看了一圈，发现已经有了不少进展，于是直接宣布“项目完成了”。

这意味着问题实际上可以拆成两个部分。第一，我们需要在项目一开始就搭建出一个环境，为用户 prompt 所要求的功能提供基础支架，让 agent 能够以“一步一步、一个功能一个功能”的方式推进。第二，我们需要 prompt 每一轮 agent：既要让它朝着目标做增量进展，也要在 session 结束时把环境留在一个干净状态。这里所说的“干净状态”，指的是一种适合直接合并到主分支的代码状态：没有严重 bug，代码组织有序、文档清晰，总之，任何开发者都可以直接继续做新功能，而不必先花时间收拾无关的烂摊子。

在内部实验中，我们通过一个双组件方案来解决这些问题：

- `Initializer agent`：第一个 agent session 使用专门的 prompt，让模型搭建初始环境，包括一个 `init.sh` 脚本、一个记录历代 agent 工作内容的 `claude-progress.txt` 文件，以及一个最初的 git commit，用于标明新增了哪些文件。
- `Coding agent`：之后的每个 session 都要求模型只做增量进展，并在结束时留下结构化更新。<sup>1</sup>

这里最关键的洞见是：必须让 agent 在进入一个全新上下文窗口时，能够快速理解当前工作状态。这个能力主要通过 `claude-progress.txt` 文件与 git 历史共同实现。我们之所以想到这些做法，也是因为它们本身就接近优秀软件工程师的日常工作方式。

## 环境管理

在更新后的 Claude 4 prompting guide 中，我们分享了一些面向多上下文窗口工作流的最佳实践，其中包括一种 harness 结构：`对第一个上下文窗口使用不同的 prompt`。这个“不同的 prompt”会要求 initializer agent 为后续 coding agents 搭建一个包含全部必要上下文的初始环境。在这里，我们进一步展开介绍这种环境中的关键组成部分。

### 功能列表

为了应对 agent 试图一次做完整个应用，或者过早宣布项目完成的问题，我们让 initializer agent 根据用户原始 prompt 扩展出一个全面的功能需求文件。在 claude.ai 克隆示例中，这意味着需要列出 200 多个功能点，例如：“用户可以打开一个新聊天，输入查询，按回车，并看到 AI 响应。”这些功能在一开始都被标记为“失败”，这样后续的 coding agents 就能清楚知道什么才算是完整功能。

```json
{
  "category": "functional",
  "description": "New chat button creates a fresh conversation",
  "steps": [
    "Navigate to main interface",
    "Click the 'New Chat' button",
    "Verify a new conversation is created",
    "Check that chat area shows welcome state",
    "Verify conversation appears in sidebar"
  ],
  "passes": false
}
```

我们会 prompt coding agents：它们只能通过修改 `passes` 字段来编辑这个文件，并使用很强烈的措辞，例如“绝对不能删除或修改测试，因为这会导致功能缺失或引入 bug。”经过一些实验，我们最终决定使用 JSON 来存储功能列表，因为相比 Markdown，模型更不容易不恰当地改写或覆盖 JSON 文件。

### 增量进展

有了上述初始环境支架之后，下一版 coding agent 就被要求：一次只做一个功能。这种增量式工作方式，对于解决 agent 想“一口气做太多”的倾向至关重要。

但即便采用增量方式，模型在改完代码后仍然必须把环境留在一个干净状态。我们的实验发现，要让模型稳定做到这一点，最有效的方法是要求它把进展提交到 git，并使用描述性的 commit message，同时把进展总结写入 progress 文件。这样，模型就可以借助 git 回滚糟糕的修改，并恢复到代码库的可工作状态。

这些做法也提高了效率，因为它们消除了 agent 对“之前到底发生了什么”的猜测需求，避免其把时间浪费在重新修复基础应用上。

### 测试

我们观察到的最后一个主要失败模式，是 Claude 倾向于在没有充分测试的情况下，就把某个功能标记为完成。如果没有明确 prompt，Claude 往往会进行代码修改，也可能用单元测试或对开发服务器发 curl 请求来测试，但它经常无法意识到：这个功能实际上并没有端到端地真正工作。

在构建 Web 应用的场景中，一旦我们明确 prompt 它去使用浏览器自动化工具，并像真人用户一样进行完整测试，Claude 的表现就会明显变好。

Claude 通过 Puppeteer MCP server 在测试 claude.ai 克隆版时截取的截图，帮助它识别和修复那些仅从代码里看不出来的 bug。

给 Claude 提供这类测试工具后，表现有了明显改善，因为 agent 能识别并修复那些代码本身不容易暴露的问题。

当然，仍然有一些局限存在，例如 Claude 的视觉能力和浏览器自动化工具本身的限制，使它无法识别所有类型的 bug。比如，Claude 无法通过 Puppeteer MCP 看见浏览器原生的 alert modal，因此依赖这类 modal 的功能往往更容易出错。

## 快速进入工作状态

有了以上基础后，每个 coding agent 在进入一个 session 时，都会被 prompt 按照一组步骤先“搞清楚自己在哪儿”，其中一些步骤相当基础，但仍然很有帮助：

- 运行 `pwd`，确认你当前工作的目录。你只能编辑这个目录中的文件。
- 读取 git log 和 progress 文件，了解最近做过什么。
- 读取功能列表文件，选择当前最高优先级、且尚未完成的功能来做。

这种方式还能在每个 session 里帮 Claude 节省一些 tokens，因为它不需要重新摸索代码该怎么测试。我们还会让 initializer agent 写一个 `init.sh` 脚本，用来启动开发服务器，并在实现新功能之前先跑一遍基础端到端测试。

在 claude.ai 克隆案例中，这意味着 agent 总是会先启动本地开发服务器，再使用 Puppeteer MCP 新建一个对话、发送一条消息，并收到一条响应。这样一来，如果应用被留在了损坏状态，Claude 就能立刻发现并修复已有 bug；如果它直接去做新功能，情况往往只会更糟。

在这种设置下，一个典型 session 的开头大致会是这样：

```text
[Assistant] I'll start by getting my bearings and understanding the current state of the project.
[Tool Use] <bash - pwd>
[Tool Use] <read - claude-progress.txt>
[Tool Use] <read - feature_list.json>
[Assistant] Let me check the git log to see recent work.
[Tool Use] <bash - git log --oneline -20>
[Assistant] Now let me check if there's an init.sh script to restart the servers.
<Starts the development server>
[Assistant] Excellent! Now let me navigate to the application and verify that some fundamental features are still working.
<Tests basic functionality>
[Assistant] Based on my verification testing, I can see that the fundamental functionality is working well. The core chat features, theme switching, conversation loading, and error handling are all functioning correctly. Now let me review the tests.json file more comprehensively to understand what needs to be implemented next.
<Starts work on a new feature>
```

## Agent 的失败模式与解决方案

| 问题 | Initializer Agent 行为 | Coding Agent 行为 |
|---|---|---|
| Claude 过早宣布整个项目完成 | 建立功能列表文件：根据输入规范，建立一个结构化 JSON 文件，列出端到端功能描述 | 在 session 开头读取功能列表文件，选择一个功能开始工作 |
| Claude 让环境处于带 bug 或进展未记录的状态 | 写入初始 git 仓库和进展记录文件 | 在 session 开头读取进展文件与 git log，并对开发服务器跑一个基础测试以捕捉未记录 bug；在 session 结尾写入 git commit 和进展更新 |
| Claude 过早把功能标记为完成 | 建立功能列表文件 | 对每个功能做自我验证；只有经过仔细测试后才把功能标记为 “passing” |
| Claude 需要花时间摸索如何运行应用 | 写一个可启动开发服务器的 `init.sh` 脚本 | 在 session 开头读取 `init.sh` |

这张表总结了长时间运行 AI agent 的四种常见失败模式及其对应解决方案。

## 未来工作

这项研究展示了一种可能的长时间运行 agent harness 设计，用来帮助模型在多个上下文窗口之间持续做增量进展。不过，依然有很多开放问题存在。

最值得关注的一点是：目前还不清楚，一个单一、通用的 coding agent，是否真的是跨多个上下文最优的选择；也许，多 agent 架构能做得更好。比如，一个专门测试的 agent、一个质量保障 agent、或者一个代码清理 agent，都可能在软件开发生命周期的某些子任务上表现得更出色。

此外，这个 demo 是围绕全栈 Web 应用开发优化的。未来一个重要方向，是把这些经验推广到其他领域。很可能，其中一部分甚至全部经验，都能迁移到其他长时 agent 任务中，例如科学研究或金融建模。

## 致谢

本文由 Justin Young 撰写。特别感谢 David Hershey、Prithvi Rajasakeran、Jeremy Hadfield、Naia Bouscal、Michael Tingley、Jesse Mu、Jake Eaton、Marius Buleandara、Maggie Vo、Pedram Navid、Nadine Yasser 和 Alex Notov 的贡献。

这项工作体现了 Anthropic 内多个团队的集体努力，正是这些团队的工作，使 Claude 能够安全地进行长时自主软件工程，尤其是 code RL 和 Claude Code 团队。欢迎有兴趣参与这项工作的候选人前往 `anthropic.com/careers` 申请。

<sup>1</sup> 原文此处为脚注说明，指后续每个 session 中的 coding agent 都被要求留下结构化更新。
