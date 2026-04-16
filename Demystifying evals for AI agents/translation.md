# Demystifying evals for AI agents - 中文翻译

发布时间：`2026-01-09`

让 AI agent 变得有用的那些能力，也恰恰让它更难被评估。能够在真实部署中发挥作用的评估策略，通常都要与被评估系统的复杂度相匹配。

## 引言

好的评估能帮助团队更有信心地发布 AI agent。没有评估时，团队很容易陷入一种被动循环：问题只有到了生产环境里才会暴露，而修掉一个失败点，往往又会引入新的失败。`Evals` 能在问题影响用户之前，把问题和行为变化显性化，而且它的价值会随着 agent 的生命周期不断累积。

正如我们在《Building effective agents》中提到的，agent 会跨越很多轮进行工作：调用工具、修改状态、并根据中间结果不断调整。也正因为如此，那些让 AI agent 有用的能力，比如自主性、智能性和灵活性，也让它们更难评估。

通过内部实践，以及和处在 agent 开发前沿的客户合作，我们逐渐摸索出一套更严谨、也更有用的 agent 评估方法。下面这些经验，已经在多种 agent 架构和真实部署场景中被证明是有效的。

## 评估的结构

一个 evaluation（`eval`）本质上是对 AI 系统的一次测试：给它一个输入，再用一套评分逻辑去检查输出是否成功。本文重点讨论的是自动化评估，也就是那些能在开发阶段运行、而不需要真实用户参与的 eval。

单轮评估相对简单：一个 prompt，一个响应，再加一段评分逻辑。对于更早期的 LLM，单轮、非 agent 型 eval 曾是最主要的评估方式。随着 AI 能力不断增强，多轮评估也越来越常见。

在一个简单 eval 里，agent 接收 prompt，grader 检查输出是否符合预期。更复杂的多轮 eval 则不同，比如一个 coding agent 会拿到工具、任务（这里的示例是构建一个 MCP server）和运行环境，然后执行一个“agent loop”（也就是工具调用和推理），并在环境中完成实现。最终，再通过单元测试验证这个 MCP server 是否真的可用。

而 agent eval 又比这更复杂。Agent 会在很多轮里使用工具、改变环境状态，并边做边调整策略，这意味着错误可能会传播、叠加。前沿模型甚至还可能找到超出静态评估预期的创造性解法。比如，Opus 4.5 在一个关于预订航班的 `τ2-bench` 任务里，发现了策略中的一个漏洞。按照原来的 eval 写法，它“失败”了，但实际上它给用户找到了一个更优解。

在构建 agent eval 时，我们使用以下定义：

- 一个 `task`（也叫 problem 或 test case）是一个单独的测试，具有明确的输入和成功标准。
- 每次对 task 的尝试叫做一个 `trial`。由于模型的输出在不同运行之间会变化，我们通常会运行多个 trial 来获得更稳定的结果。
- `Grader` 是一段给 agent 表现某个方面打分的逻辑。一个 task 可以有多个 grader，而每个 grader 里又可以包含多个 assertion（有时也叫 check）。
- `Transcript`（也叫 trace 或 trajectory）是某次 trial 的完整记录，包括输出、工具调用、推理、中间结果，以及其他所有交互。对于 Anthropic API 来说，就是一次 eval 结束时完整的 `messages` 数组，里面包含评估过程中所有 API 调用和所有返回结果。
- `Outcome` 指 trial 结束时环境里的最终状态。一个订机票 agent 可能在 transcript 结尾说“你的航班已经预订成功”，但真正的 outcome 是环境里的 SQL 数据库中是否真的存在这条预订记录。
- `Evaluation harness` 是把 eval 从头到尾跑起来的基础设施。它负责提供指令和工具、并发运行任务、记录每一步、给输出打分，并汇总结果。
- `Agent harness`（或 scaffold）是让模型能以 agent 身份行动的系统：它负责处理输入、编排工具调用、返回结果。我们评估“一个 agent”时，评估的其实是 harness 和模型协同工作的整体。比如 Claude Code 就是一个灵活的 agent harness，而我们又通过 Agent SDK 中的核心原语，构建了长时运行的 agent harness。
- `Evaluation suite` 是一组 task 的集合，用来衡量特定能力或行为。一个 suite 中的 task 往往共享一个宽泛目标。比如一个客服 eval suite 可能会覆盖退款、取消订单和升级处理等场景。

## 为什么要做评估？

很多团队刚开始构建 agent 时，往往只靠手动测试、内部试用和经验判断，也能推进得相当快。更严格的评估机制甚至会让人觉得是一种拖慢发布速度的额外负担。但在早期原型阶段过去之后，一旦 agent 进入生产并开始规模化，仅靠这种方式开发就会越来越吃力。

真正的崩点，通常出现在用户开始反馈“改完之后 agent 变差了”，而团队却像“闭着眼开车”一样，除了靠猜测和反复试错，没有别的方法来验证。没有 eval 时，调试就是被动的：等用户抱怨、人工复现、修 bug、然后祈祷别的地方没回退。团队无法分辨真实回退和随机噪声，无法在发布前把改动自动测试到几百种场景上，也无法量化改进。

我们已经很多次看到这种演进路径。比如 Claude Code 早期就是依靠 Anthropic 员工和外部用户的反馈快速迭代。后来我们引入了 eval，先覆盖一些窄场景，比如简洁性和文件编辑，再慢慢扩展到更复杂的行为，比如过度设计。它们帮助我们识别问题、指导改进，并让研究团队和产品团队之间的协作更聚焦。结合生产监控、A/B 测试、用户研究等方法，eval 为 Claude Code 的持续优化提供了稳定信号。

无论 agent 处于生命周期的哪个阶段，写 eval 都有价值。早期，它会迫使产品团队明确“成功”到底意味着什么；后期，它则帮助团队维持稳定的一致质量线。

Descript 的 agent 帮助用户编辑视频，因此他们围绕一个成功编辑流程的三个维度建立 eval：不要把东西弄坏、按用户要求去做、并且做得好。他们先从人工评分开始，后来逐步发展到由产品团队定义标准、再定期用人工校准的 LLM grader，现在会定期运行两套 eval：一套做质量基准，一套做回归测试。Bolt AI 团队则是到了更晚阶段才开始系统化构建 eval，当时他们的 agent 已经被广泛使用。3 个月内，他们搭建出一套评估系统：运行 agent、用静态分析给输出打分、用 browser agents 测试应用，并使用 LLM judges 检查诸如指令遵循等行为。

有些团队在开发一开始就做 eval；有些团队则是在系统已经扩大到一定规模、eval 成了继续优化 agent 的瓶颈时才补上。实际上，在 agent 开发一开始做 eval 特别有价值，因为它能把预期行为显式编码下来。两个工程师看同一份初始 spec，可能会对 AI 应该怎样处理边缘情况得出不同理解；而 eval suite 会把这种歧义消掉。无论它是在什么时候创建的，eval 都能加速开发。

评估还会直接影响你采用新模型的速度。更强的新模型发布时，没有 eval 的团队往往要花几周手工测试；而有 eval 的团队可以很快判断模型的优势、调 prompt，并在几天内完成升级。

一旦 eval 建好了，基线和回归测试几乎就是“免费”的：延迟、token 用量、单任务成本、错误率等指标，都可以在固定任务集上持续追踪。Eval 还可能成为产品团队与研究团队之间带宽最高的沟通渠道，因为它给出了研究人员可以直接优化的指标。可以看到，eval 的价值远远不止“跟踪回退和改进”；它的复利效应很容易被低估，因为成本是在前期可见地支出，而收益则是在后期不断累积。

## 如何评估 AI agents

我们今天看到的常见大规模部署 agent，大体包括几类：coding agents、research agents、computer use agents，以及 conversational agents。它们可能落在完全不同的行业里，但评估方法其实有不少共通之处。你不需要从零发明一套评估体系。下面这些部分总结了几种成熟且被反复验证的做法，你可以先把它们当成基础，再扩展到自己的领域。

## Agent 评估中的 grader 类型

Agent eval 通常会组合三类 grader：基于代码的 grader、基于模型的 grader，以及人工 grader。每一种 grader 都会评估 transcript 或 outcome 的某一部分。有效的 eval 设计中，一个核心问题就是为任务选对 grader。

### 基于代码的 graders

| 方法 | 优势 | 弱点 |
| --- | --- | --- |
| 字符串匹配检查（精确匹配、正则、模糊匹配等） | 快 | 对合法变体很脆弱，只要和预期模式不完全一致就可能误判 |
| 二元测试（`fail-to-pass`、`pass-to-pass`） | 便宜 | 缺乏细腻度 |
| 静态分析（lint、type、security） | 客观 | 对一些更主观的任务能力有限 |
| 结果验证 | 可复现 |  |
| 工具调用验证（用了哪些工具、参数是否正确） | 容易调试 |  |
| Transcript 分析（轮数、token 用量） | 能验证特定条件 |  |

### 基于模型的 graders

| 方法 | 优势 | 弱点 |
| --- | --- | --- |
| 基于 rubric 的打分 | 灵活 | 非确定性 |
| 自然语言断言 | 可扩展 | 比代码 grader 更贵 |
| 成对比较 | 能捕捉细微差别 | 需要用人工 grader 做校准，才能保证准确性 |
| 基于参考答案的评估 | 能处理开放式任务 |  |
| 多评委共识 | 能处理自由格式输出 |  |

### 人工 graders

| 方法 | 优势 | 弱点 |
| --- | --- | --- |
| 领域专家评审（`SME review`） | 质量金标准 | 昂贵 |
| 众包判断 | 更接近专家用户的真实判断 | 慢 |
| 抽样 spot-check | 可用于校准模型 grader | 往往需要大量可调用的人类专家 |
| A/B 测试 |  |  |
| 标注者一致性（`inter-annotator agreement`） |  |  |

对于每个 task，最终评分方式可以是加权式（多个 grader 的组合得分必须达到阈值）、二元式（所有 grader 都必须通过），或混合式。

## 能力型 eval 与回归型 eval

能力型或“质量型” eval 关注的是：“这个 agent 能把什么事情做好？” 这类 eval 一开始的通过率应该比较低，目标是覆盖 agent 目前还不擅长的任务，给团队一个明确的“爬坡目标”。

回归型 eval 关注的是：“这个 agent 现在是否仍然能处理它过去能处理的所有任务？” 它的通过率应当接近 100%。它的作用是防止倒退，只要得分下降，就说明有什么地方坏了，需要修复。团队在能力型 eval 上不断爬坡时，也必须同步跑回归型 eval，以确保改动不会在别处引发问题。

一个 agent 发布并优化一段时间之后，那些原本通过率已经很高的能力型 eval，往往会“毕业”成回归套件，持续运行以捕捉系统漂移。原来用于回答“我们到底能不能做成这件事”的任务，之后就会转而回答“我们现在还能不能稳定做好这件事”。

## 评估 coding agents

Coding agent 会像人类开发者那样写代码、测试代码、调试代码，浏览代码库并运行命令。对于现代 coding agent，最有效的 eval 通常依赖三件事：定义明确的任务、稳定的测试环境，以及对生成代码进行充分验证的测试集。

确定性 grader 天然适合 coding agent，因为软件通常很容易评估：代码能不能运行？测试能不能通过？两个被广泛使用的 coding agent benchmark，`SWE-bench Verified` 和 `Terminal-Bench`，都采用了这一思路。`SWE-bench Verified` 会给 agent 来自知名 Python 仓库的 GitHub issue，然后通过运行测试套件来评估解决方案；只有在修复失败测试且不破坏已有测试的前提下，方案才算通过。LLM 在这个 eval 上的成绩，仅仅一年就从 40% 提升到了 80% 以上。`Terminal-Bench` 走的是另一条路线：它测试端到端技术任务，比如从源码构建 Linux 内核，或训练一个机器学习模型。

一旦你已经有了一组可以验证 coding task 关键结果的 pass-or-fail 测试，通常也很值得继续对 transcript 做评分。比如，可以用基于启发式的代码质量规则，从“是否通过测试”以外的角度评估生成代码；也可以配合清晰 rubric 的模型 grader，评估 agent 是如何调用工具、如何与用户互动等行为。

### 示例：一个 coding agent 的理论评估

设想这样一个编码任务：agent 必须修复一个身份认证绕过漏洞。如下所示的 YAML 文件只是一个说明性例子，展示了如何同时使用 grader 和 metrics 来评估这个 agent。

```yaml
task:
  id: "fix-auth-bypass_1"
  desc: "Fix authentication bypass when password field is empty and ..."
  graders:
    - type: deterministic_tests
      required: [test_empty_pw_rejected.py, test_null_pw_rejected.py]
    - type: llm_rubric
      rubric: prompts/code_quality.md
    - type: static_analysis
      commands: [ruff, mypy, bandit]
    - type: state_check
      expect:
        security_logs: {event_type: "auth_blocked"}
    - type: tool_calls
      required:
        - {tool: read_file, params: {path: "src/auth/*"}}
        - {tool: edit_file}
        - {tool: run_tests}
  tracked_metrics:
    - type: transcript
      metrics:
        - n_turns
        - n_toolcalls
        - n_total_tokens
    - type: latency
      metrics:
        - time_to_first_token
        - output_tokens_per_sec
        - time_to_last_token
```

需要注意的是，这个例子为了说明方便，展示了几乎所有可用 grader 的范围。实际中，coding eval 往往主要依赖单元测试来验证正确性，再加一个 LLM rubric 用于整体代码质量评估；其他 grader 和指标通常只在有明确需要时才添加。

## 评估 conversational agents

Conversational agent 常见于客服、销售、教练等场景。与传统 chatbot 不同，这类 agent 会维护状态、使用工具，并在对话过程中执行动作。虽然 coding agent 和 research agent 也可能和用户进行很多轮交互，但 conversational agent 有一个独特挑战：交互质量本身就是评估对象的一部分。对它们来说，有效的 eval 通常依赖两类东西：可验证的终态 outcome，以及能同时衡量任务完成情况和互动质量的 rubric。与大多数其他 eval 不同，这类评估往往还需要另一个 LLM 来模拟用户。我们在 alignment auditing agents 中就使用了这种方法，通过长时间、对抗性的对话来对模型做压力测试。

Conversational agent 的成功往往是多维度的：工单是否解决了（state check）？是否在 10 轮以内完成（transcript constraint）？语气是否合适（LLM rubric）？`τ-Bench` 及其后继 `τ2-Bench` 就是体现这种多维评估的两个 benchmark。它们在零售客服、航空订票等领域中模拟多轮互动，让一个模型扮演用户画像，而 agent 则在真实场景中推进任务。

### 示例：一个 conversational agent 的理论评估

假设有这样一个客服任务：agent 需要处理一位情绪很差用户的退款请求。

```yaml
graders:
  - type: llm_rubric
    rubric: prompts/support_quality.md
    assertions:
      - "Agent showed empathy for customer's frustration"
      - "Resolution was clearly explained"
      - "Agent's response grounded in fetch_policy tool results"
  - type: state_check
    expect:
      tickets: {status: resolved}
      refunds: {status: processed}
  - type: tool_calls
    required:
      - {tool: verify_identity}
      - {tool: process_refund, params: {amount: "<=100"}}
      - {tool: send_confirmation}
  - type: transcript
    max_turns: 10
tracked_metrics:
  - type: transcript
    metrics:
      - n_turns
      - n_toolcalls
      - n_total_tokens
  - type: latency
    metrics:
      - time_to_first_token
      - output_tokens_per_sec
      - time_to_last_token
```

和前面的 coding agent 示例一样，这个任务也只是为了展示多种 grader 的组合方式。实际中，conversational agent 的评估通常主要使用模型 grader 来同时判断沟通质量和目标完成度，因为很多任务，例如回答某个问题，本来就可能存在不止一种“正确解”。

## 评估 research agents

Research agent 会去搜集、综合和分析信息，然后产出答案或报告。与 coding agent 能用单元测试给出二元通过/失败信号不同，研究质量只能相对于任务本身来判断。什么叫“足够全面”“来源充分”甚至“正确”，都取决于上下文。市场扫描、并购尽调、科学报告，这些任务所需要的标准都完全不同。

Research eval 面临一些独特挑战：专家之间可能会对一份综合是否足够全面产生分歧；参考内容不断变化，因此 ground truth 也在漂移；输出越长、越开放，犯错空间就越大。比如 `BrowseComp` 这种 benchmark，会测试 AI agent 是否能在开放网页环境中“大海捞针”地找到答案，这些问题都设计成“容易验证、但很难解决”。

构建 research agent eval 的一种策略，是组合多种 grader。`Groundedness` 检查用来确认结论是否有检索到的来源支撑；`coverage` 检查用来定义一个好答案必须包含哪些关键事实；`source quality` 检查则确认 agent 引用的是权威来源，而不只是第一个搜到的页面。对于那些客观上有确定答案的任务，比如“Company X 的 Q3 revenue 是多少？”，精确匹配也适用。LLM 既可以帮助标出无依据的陈述和覆盖缺口，也可以对开放式综合的连贯性和完整性做判断。

也正因为 research 质量带有明显主观性，基于 LLM 的 rubric 必须频繁和专家人工判断对齐校准，才能真正有效地评估这类 agent。

## Computer use agents

Computer use agent 通过与人类相同的界面来操作软件，比如截图、鼠标点击、键盘输入和滚动，而不是通过 API 或执行代码。它们几乎可以使用任何带图形界面（GUI）的应用，从设计工具到遗留企业软件都行。对这类 agent 的评估，需要在一个真实或沙盒环境中运行 agent，让它操作软件应用，然后再检查它是否达成了预期结果。比如 `WebArena` 测试浏览器任务时，会通过 URL 和页面状态检查 agent 是否正确导航；对那些会修改数据的任务，则进一步检查后端状态，确认订单是否真的下单成功，而不只是停留在确认页。`OSWorld` 则把这类评估扩展到了完整操作系统控制，它会在任务结束后运行评估脚本，检查多种工件，包括文件系统状态、应用配置、数据库内容和 UI 元素属性等。

Browser use agent 在 token 效率和延迟之间需要取得平衡。基于 DOM 的交互执行很快，但会消耗很多 token；基于截图的交互更慢，却往往更省 token。举个例子，如果让 Claude 总结 Wikipedia 页面，直接从 DOM 提取文本更有效；但如果让它去 Amazon 上找一个新的 laptop case，则截屏往往更划算，因为整页 DOM 太长、token 成本太高。在我们的 `Claude for Chrome` 产品中，我们专门开发了 eval，检查 agent 是否在不同上下文里选择了正确工具，这帮助我们更快、更准确地完成浏览器任务。

## 如何理解 agent eval 中的非确定性

不论是哪种 agent，agent 的行为在不同运行之间都会变化，这让 eval 结果比看起来更难解释。每个 task 都有自己的成功率，也许某个 task 是 90%，另一个是 50%；同一个 task 这次通过了，下次也可能失败。有时，我们真正想衡量的是：一个 agent 在某个 task 上究竟有多大概率成功。

两个指标能更好地表达这种差异：

`pass@k` 衡量的是：agent 在 `k` 次尝试中至少得到一次正确解的概率。随着 `k` 增大，`pass@k` 会上升，因为“多几次射门”意味着更高概率至少命中一次。比如 `50% pass@1` 表示模型在 eval 中有一半任务能在第一次尝试时成功。在 coding 场景中，我们通常最关心的是 agent 能否第一次就找到解，也就是 `pass@1`。但在另一些场景里，只要多种方案里有一个可行，提出很多候选方案本身也是合理的。

`pass^k` 衡量的是：`k` 次 trial 全部成功的概率。随着 `k` 增大，`pass^k` 反而会下降，因为要求在更多轮里持续稳定成功，本来就更难。如果你的 agent 单次 trial 成功率是 75%，跑 3 次 trial，那么三次都通过的概率就是 `(0.75)^3 ≈ 42%`。这个指标对面向用户的 agent 尤其重要，因为用户期待的是“每次都可靠”。

随着 trial 数量增加，`pass@k` 和 `pass^k` 会明显分化。`k=1` 时它们相同，二者都等于单次 trial 成功率；但到了 `k=10`，它们会讲述完全相反的故事：`pass@k` 接近 100%，而 `pass^k` 则会跌向 0。

这两个指标都很有用，用哪一个要看产品要求：如果“一次成功就够了”，用 `pass@k`；如果“稳定一致性”才重要，就看 `pass^k`。

## 从零到一：构建可靠 agent eval 的路线图

这一部分总结的是我们在真实场景中验证过的实操建议，帮助团队从“完全没有 eval”走到“拥有值得信赖的 eval”。你可以把它理解成 eval-driven agent development 的路线图：尽早定义成功、清晰衡量成功，并持续迭代。

## 为初始 eval 数据集收集任务

### Step 0. 尽早开始

我们经常看到团队因为觉得“至少得先准备几百个 task”而推迟构建 eval。实际上，20 到 50 个简单 task，哪怕只是来自真实失败案例，也已经是一个很好的起点。毕竟在 agent 开发早期，系统每一次改动通常都会带来肉眼可见的变化，效应量很大，所以小样本也足够。更成熟的 agent 也许确实需要更大的、更难的 eval 才能检测较小改进，但一开始最好的方式仍然是先走 80/20。Eval 越晚开始越难做。早期时，产品需求还很自然地能转成测试用例；拖得越久，你越像是在从一个已经上线的系统里“反向推导”成功标准。

### Step 1. 先从你已经在手动测试的内容开始

从开发过程中本来就在人工检查的行为开始，也就是每次发布前都要确认的东西，以及最终用户最常尝试的那些任务。如果你已经上线了，那就去看 bug tracker 和支持队列。把用户报告过的失败直接转成测试用例，能确保你的 suite 反映真实使用情况；再按用户影响程度排优先级，能帮助你把精力花在真正重要的地方。

### Step 2. 写出无歧义的任务，并准备参考解

任务质量比看起来更难把握。一个好的 task，应该满足这样一个条件：两个领域专家独立判断，也会得出同样的通过/失败结论。如果连他们自己都无法顺利完成这个 task，那它就还需要打磨。任务定义里的歧义会直接变成指标噪声。模型 grader 的标准也是一样，rubric 写得模糊，判断就会不一致。

每个 task 都应该是一个“按说明正确执行的 agent 可以完成的任务”。这一点有时很微妙。比如在 `Terminal-Bench` 的审计中我们发现：如果 task 要求 agent 写一个脚本，却没有明确 filepath，而测试却假设脚本必须位于某个固定路径，那 agent 就会在自己并没有做错的情况下失败。也就是说，grader 检查的每一个条件，都应该能从 task 描述中清楚推出；agent 不该因为规格模糊而失败。对于前沿模型而言，如果很多 trial 下某个 task 的通过率始终是 0%，也就是 `0% pass@100`，那通常更可能说明 task 自身坏了，而不是 agent 没能力。此时最应该先检查的是 task 规格和 grader。对每个 task，准备一个参考解都很有帮助，也就是一个已知可行、并能通过全部 grader 的输出。它既能证明任务可解，也能验证 grader 配置无误。

### Step 3. 构建平衡的问题集

不仅要测试“某行为应该发生”的情况，也要测试“它不应该发生”的情况。单边 eval 会导致单边优化。比如，如果你只测 agent 在“应该搜索时是否会搜索”，最后很可能得到一个“几乎什么都去搜”的 agent。尽量避免类别分布严重失衡的 eval。我们在为 Claude.ai 的 web search 功能构建 eval 时，就亲身经历过这一点。难点在于既要防止模型在不该搜的时候去搜，又要保留它在该做深入研究时进行充分搜索的能力。团队最后构建了一套双向覆盖的 eval：既包含模型应当搜索的查询，比如查天气；也包含模型应当依靠已有知识直接回答的查询，比如“Apple 是谁创立的？” 如何在 `undertriggering`（该搜不搜）和 `overtriggering`（不该搜却去搜）之间取得平衡，非常困难，也需要对 prompt 和 eval 进行很多轮迭代。随着更多真实案例出现，我们也会不断把它们补充进 eval，提升覆盖度。

## 设计 eval harness 和 graders

### Step 4. 构建稳健的 eval harness 和稳定环境

非常重要的一点是：eval 中运行的 agent，应该和生产环境中的 agent 大体一致；同时，环境本身不能引入额外噪声。每个 trial 都应当从一个干净环境开始，以保证“隔离”。任何不必要的共享状态，比如残留文件、缓存数据、资源耗尽，都会让失败相互关联，结果反映的将是基础设施抖动，而不是 agent 的真实能力。共享状态甚至还可能虚假抬高成绩。比如在一些内部 eval 中，我们观察到 Claude 会通过查看前一次 trial 的 git history，给自己带来不公平优势。如果多个本应独立的 trial 因为同一环境限制而同时失败，比如 CPU 内存不足，那么它们就不再独立，eval 结果也就不能可靠衡量 agent 表现。

### Step 5. 认真设计 graders

正如前面所说，优秀的 eval 设计，核心是为具体 agent 和任务选对 grader。我们的建议是：能用确定性 grader 的地方就优先用；只有在必要时，或为了获得额外灵活性时，再使用 LLM grader；人工 grader 则应审慎使用，主要作为补充验证。

一个常见冲动是，去检查 agent 是否严格按某条固定路径执行，例如是否按照正确顺序调用了某几个工具。我们的经验是，这种做法太僵硬，最终只会得到极度脆弱的测试，因为 agent 往往会找到评估设计者事先没想到、但同样合理的完成路径。为了不无谓惩罚创造性，通常更好的做法是评估 agent 最终产出了什么，而不是它走了哪条路径。

如果任务本身包含多个组成部分，最好设计出部分得分机制。一个客服 agent 如果正确识别了问题，也完成了身份验证，但最后退款没处理成功，它显然仍然比一个一开始就彻底失败的 agent 更好。Eval 结果里应该反映这种成功连续体。

模型评分往往需要反复迭代，才能确认准确性。`LLM-as-judge` grader 应与人类专家密切校准，直到你对人工评分与模型评分之间几乎没有显著偏差有足够信心。为了减少 hallucination，最好在指令里给模型留出“退路”，例如允许它在信息不足时返回 `Unknown`。另一个有用做法是：为任务的每个维度都定义清晰结构化 rubric，然后让不同的 LLM judge 分别给各维度打分，而不是让一个 judge 一口气评完整个任务。等系统稳定之后，再偶尔用人工复核即可。

有些 eval 会存在很隐蔽的失败模式，导致即使 agent 表现不错，最终分数仍然偏低，原因可能是 grader bug、agent harness 限制，或者任务本身含糊。即使是很成熟的团队，也会漏掉这些问题。比如，Opus 4.5 起初在 `CORE-Bench` 上只有 42%，后来 Anthropic 研究员发现了多个问题：评分逻辑太死板，期待值是 `96.124991...` 时，却把 `96.12` 判错；任务描述存在歧义；还有一些任务本身带随机性，根本无法精确复现。修掉这些 bug、换成限制更少的 scaffold 后，Opus 4.5 的分数跃升到了 95%。类似地，METR 也发现他们的 time horizon benchmark 里，有些任务配置错误：任务说明要求 agent 优化到某个给定阈值，但评分逻辑却要求超过这个阈值。结果像 Claude 这样认真按要求做的模型反而吃亏，而忽视说明的模型却得分更高。认真复查 task 和 grader，能帮助你避免这些问题。

最后，要让 grader 对“钻空子”或“投机取巧”有足够抵抗力。Agent 不应该轻易“作弊”通过 eval。Task 和 grader 的设计，应当确保通过意味着它真的解决了问题，而不是利用了某个意外漏洞。

## 长期维护和使用 eval

### Step 6. 去读 transcripts

如果你不去看大量 trial 的 transcript 和评分结果，就无法真正知道 grader 是否工作正常。在 Anthropic，我们投入了专门工具来查看 eval transcript，也会定期花时间阅读它们。某个 task 失败时，transcript 会告诉你：是 agent 真的犯错了，还是 grader 把一个合法方案误判成失败。它往往也会暴露出很多关于 agent 与 eval 本身行为的重要细节。

失败看起来必须“公平”：你应该能清楚看出 agent 错在哪里、为什么错。如果分数上不去，我们必须有信心确认这是 agent 表现问题，而不是 eval 自己的问题。阅读 transcript，是确认 eval 是否真的在衡量关键问题的唯一办法，也是 agent 开发中的核心技能。

### Step 7. 监控能力型 eval 是否饱和

一个 100% 的 eval 虽然能追踪回归，但对继续提升已经没有信号。所谓 eval saturation，就是 agent 已经通过了所有可解任务，导致这个 eval 不再有改进空间。比如 `SWE-Bench Verified` 今年初的成绩大约是 30%，而前沿模型现在已经逼近 80% 以上，开始接近饱和。随着 eval 趋于饱和，进步也会看起来放缓，因为剩下的都是最难的任务。这会让结果产生误导：实际上能力提升很大，但分数只涨了一点点。例如，代码审查创业公司 Qodo 起初对 Opus 4.5 并不惊艳，因为他们的一次性 coding eval 无法反映在更长、更复杂任务上的提升。后来他们开发了新的 agentic eval framework，这才更清楚地看到了真实进展。

作为一条经验法则，在有人深入研究 eval 细节并读过一部分 transcript 之前，我们不会轻信任何 eval 分数。如果评分不公平、任务有歧义、合法解法被惩罚，或者 harness 限制了模型，那就应该修订 eval。

### Step 8. 通过开放贡献和持续维护，让 eval suite 长期健康

Eval suite 是一个活的产物，要想持续有用，它需要持续投入，也需要清晰的责任归属。

在 Anthropic，我们尝试过多种维护 eval 的方式。最终最有效的做法是：设立专门的 eval 团队负责核心基础设施，同时由领域专家和产品团队贡献大部分 eval task，并亲自运行这些评估。

对于 AI 产品团队来说，拥有并持续迭代评估，应该像维护单元测试一样成为日常工作。团队常常会在一些 AI 功能上浪费数周时间，因为这些功能在早期测试里“看起来能用”，但实际上没有满足一些没有被说清楚的要求，而这些问题本来可以通过一个设计得当的 eval 提前暴露出来。定义 eval task，本身就是检验产品需求是否足够具体、能否开始实施的最佳方式之一。

我们建议实践 `eval-driven development`：在 agent 真正具备某项能力之前，就先用 eval 定义这项能力，然后围绕它持续迭代，直到 agent 表现良好。我们内部也经常会做一些“今天只是勉强能用，但赌几个月后模型会更强”的功能。那些一开始通过率很低的 capability eval，会把这种“赌注”显性化。等新模型发布后，跑一遍 suite，很快就知道哪些赌对了。

最接近产品需求和用户的人，最有资格去定义什么叫成功。以当前模型能力来看，产品经理、客户成功经理、销售人员，完全可以用 Claude Code 提交一个 eval task 的 PR。让他们参与进来吧，甚至更进一步，主动去赋能他们。

## eval 如何与其他方法配合，形成对 agent 的整体理解

自动化 eval 可以在不影响真实用户、也无需上线到生产环境的前提下，对 agent 运行成千上万次任务。但它只是理解 agent 表现的诸多方法之一。完整图景还需要结合生产监控、用户反馈、A/B 测试、人工 transcript 审查，以及系统化的人类评估。

| 方法 | 优点 | 缺点 |
| --- | --- | --- |
| 自动化 eval | 迭代快；完全可复现；不影响真实用户；每次提交都能跑；能在不上线的情况下大规模覆盖场景 | 需要前期投入；随着产品和模型演化需要持续维护；如果与真实使用模式不匹配，会制造虚假信心 |
| 生产监控 | 能看到真实用户在规模化环境中的行为；能抓到合成 eval 发现不了的问题；提供 agent 在真实世界中的表现真相 | 反应式；问题会先影响用户；信号噪声大；需要额外埋点；通常缺少可直接评分的 ground truth |
| A/B 测试 | 直接衡量真实用户结果（留存、任务完成率）；能控制混杂因素；可扩展且系统化 | 慢，通常要几天到几周才显著，而且需要足够流量；只能测试已部署变更；如果不细读 transcript，很难理解指标变化背后的“为什么” |
| 用户反馈 | 会暴露团队没预料到的问题；来自真实人类用户的具体案例；通常与产品目标相关 | 稀疏且自选择；偏向严重问题；用户很少解释失败原因；不可自动化；过度依赖用户发现问题会伤害用户体验 |
| 人工 transcript 审查 | 帮助建立对失败模式的直觉；能抓到自动化检查漏掉的细微质量问题；有助于校准“什么叫好” | 很耗时；不可扩展；覆盖不稳定；评审疲劳和评审差异会影响信号质量；通常更多给出定性判断，而不是清晰定量评分 |
| 系统化人工研究 | 多位人工评审给出金标准质量判断；适合主观或含糊任务；能为模型 grader 提供校准信号 | 相对昂贵、反馈慢；不适合高频执行；评审者分歧需要协调；法律、金融、医疗等复杂领域往往需要专家参与 |

这些方法适合 agent 开发的不同阶段。自动化 eval 特别适合上线前和 CI/CD 阶段，几乎可以在每次 agent 改动、每次模型升级时都跑一遍，作为抵御质量问题的第一道防线。生产监控则适合上线后，用来发现分布漂移和真实世界中的意外失败。A/B 测试适合在有足够流量后验证重大变更。用户反馈和 transcript 审查则应当持续进行，用来填补其他方法留下的空白：持续处理反馈、每周抽样阅读 transcript，并在必要时深入调查。至于系统化人工研究，则更适合用于校准 LLM grader，或评估那些只有人类共识才能作为标准的主观输出。

这很像安全工程中的“瑞士奶酪模型”：没有哪一层评估能够单独拦住所有问题；但多层方法叠加之后，从一层漏过去的失败，往往会被下一层接住。

最有效的团队，通常是把这些方法结合起来使用：自动化 eval 用于快速迭代，生产监控提供真实世界表现，周期性人工评审则提供校准。

## 结论

没有 eval 的团队，常常会陷入一种被动循环：修一个失败点，又制造另一个；分不清真正的回退和随机噪声。而那些早期就投入评估的团队，情况往往恰恰相反：随着失败被转成测试用例、测试用例再去防止回退、指标逐渐替代拍脑袋，开发会越来越快。Eval 给整个团队提供了一座清晰可见、可持续攀爬的山，把“这个 agent 好像变差了”变成一个可以采取行动的问题。它的价值会复利增长，但前提是你把 eval 当成核心组件，而不是事后补丁。

不同 agent 类型的具体模式会不同，但本文强调的基本原则是稳定的：尽早开始，不要等待完美套件；从你看到的真实失败中抽取任务；定义无歧义、足够稳健的成功标准；认真设计 grader，并组合多种类型；确保问题对模型来说足够有挑战；持续迭代评估，提高信噪比；还有一条尤其重要的建议：去读 transcript。

AI agent 评估仍然是一个新兴且快速演化的领域。随着 agent 开始承担更长时任务、在多 agent 系统中协作、并处理越来越主观的工作，我们还需要继续调整这些技术。我们也会在学习过程中持续分享最佳实践。

## 致谢

本文由 Mikaela Grace、Jeremy Hadfield、Rodrigo Olivares 和 Jiri De Jonghe 撰写。我们也感谢 David Hershey、Gian Segato、Mike Merrill、Alex Shaw、Nicholas Carlini、Ethan Dixon、Pedram Navid、Jake Eaton、Alyssa Baum、Lina Tawfik、Karen Zhou、Alexander Bricken、Sam Kennedy、Robert Ying 等人的贡献。特别感谢我们在 eval 合作中共同学习过的客户与合作伙伴，包括 iGent、Cognition、Bolt、Sierra、Vals.ai、Macroscope、PromptLayer、Stripe、Shopify、Terminal Bench 团队等。本文体现了 Anthropic 多个团队共同推动评估实践发展的集体努力。

## 附录：Eval 框架

有不少开源或商业框架，可以帮助团队在不从零搭基础设施的情况下实现 agent eval。具体该选什么，取决于你的 agent 类型、现有技术栈，以及你更需要离线评估、生产可观测性，还是两者都要。

`Harbor` 适合在容器化环境中运行 agent，提供了跨云厂商大规模运行 trial 的基础设施，也定义了统一的任务与 grader 格式。像 `Terminal-Bench 2.0` 这样的 benchmark 就通过 Harbor registry 发布，因此团队既可以直接运行成熟 benchmark，也可以把自己的自定义 eval suite 接进去。

`Braintrust` 是一个把离线评估、生产可观测性和实验追踪结合起来的平台，适合那些既要在开发中快速迭代、又要在生产中监控质量的团队。它的 `autoevals` 库内置了针对事实性、相关性等常见维度的 scorer。

`LangSmith` 提供 tracing、离线和在线评估，以及数据集管理，并和 LangChain 生态深度集成。`Langfuse` 则提供了类似能力，作为一个可自托管的开源替代方案，更适合那些对数据驻留有要求的团队。

`Arize` 提供 `Phoenix`，一个面向 LLM tracing、调试以及离线/在线 eval 的开源平台；同时也提供 `AX` 这一 SaaS 产品，用于在 Phoenix 基础上扩展规模化、优化与监控能力。

很多团队会组合使用多个工具，自己搭 eval framework，或者一开始只用简单评估脚本起步。我们的经验是，框架固然能帮助你更快推进、也更容易标准化，但它们最终有多大价值，仍然取决于你往里面放了什么样的 eval task。通常最好的做法是尽快选一个贴合工作流的框架，然后把主要精力投入到 eval 本身，也就是不断迭代高质量测试用例和 grader。
