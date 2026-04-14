# The "think" tool: Enabling Claude to stop and think in complex tool use situations - 中文翻译

发布时间：`2025-03-20`

“think” 工具：让 Claude 在复杂工具使用场景中停下来思考

一个能够提升 Claude 在复杂问题求解中表现的新工具

## Extended thinking 更新

`2025-12-15`

自最初发布以来，`extended thinking` 能力已经有了提升，因此我们现在建议在大多数场景下优先使用这一能力，而不是单独使用一个专门的 `think` 工具。`Extended thinking` 能提供类似收益，也就是给 Claude 留出空间去推理复杂问题，但它的集成方式和整体表现更好。具体实现细节请参考我们的 `extended thinking` 文档。

随着我们持续增强 Claude 处理复杂问题的能力，我们发现了一种特别有效的方法：一个名为 `think` 的工具，它能够在复杂任务过程中为结构化思考创建专门空间。

这个方法既简单又强大。正如我们下面会解释的，它与 Claude 新推出的 `extended thinking` 能力并不相同。`Think` 工具在 Claude 的 agentic 工具使用能力上带来了显著提升，包括更好地遵循策略、做出一致决策，以及处理多步骤问题，而且实现成本非常低。

在这篇文章中，我们会介绍如何在不同应用中实现 `think` 工具，并基于经过验证的基准测试结果，为开发者提供实用建议。

## 什么是 “think” 工具？

借助 `think` 工具，我们赋予 Claude 一种能力：在走向最终答案的过程中，多插入一个额外的思考步骤，并且为这个步骤提供专门的空间。

虽然它听起来和 `extended thinking` 很像，但它们是不同概念。`Extended thinking` 关注的是 Claude 在开始生成响应之前所做的事情。在 `extended thinking` 模式下，Claude 会在行动前深入考虑并反复推敲自己的计划。而 `think` 工具则是给 Claude 一个机会：在它已经开始生成响应之后，停下来思考自己是否已经拥有继续前进所需的全部信息。这一点在长链工具调用场景中，或在与用户进行长时间、多步骤对话时尤其有帮助。

因此，`think` 工具更适合这样一些场景：Claude 无法仅凭用户查询本身就形成完整响应，而是需要处理外部信息，例如工具调用结果中返回的信息。Claude 借助 `think` 工具所做的推理，通常没有 `extended thinking` 那么全面，而是更聚焦于模型在过程中发现的新信息。

我们建议，在更简单的工具使用场景中，例如非顺序型工具调用或较直接的指令遵循，优先使用 `extended thinking`。对于编码、数学、物理等不需要 Claude 调用工具的场景，`extended thinking` 同样很有帮助。而当 Claude 需要调用复杂工具、在长工具链中仔细分析工具输出、在带有大量策略约束和详细规则的环境中工作，或需要做出一系列前后相依且错误代价较高的决策时，`think` 工具会更加合适。

下面是一个使用标准工具规格格式的示例实现，来源于 `τ-Bench`：

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or change the database, but just append the thought to the log. Use it when complex reasoning or some cache memory is needed.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "A thought to think about."
      }
    },
    "required": ["thought"]
  }
}
```

## 在 τ-Bench 上的表现

我们使用 `τ-Bench` 对 `think` 工具进行了评估。`τ-Bench` 是一个综合性基准测试，旨在测试模型在真实客服场景中使用工具的能力，其中 `think` 工具本身就是评测标准环境的一部分。

`τ-Bench` 会评估 Claude 在以下方面的能力：

- 在与模拟用户的真实对话中导航
- 始终如一地遵循复杂的客服代理策略规范
- 使用多种工具访问和操作环境数据库

`τ-Bench` 使用的主要评估指标是 `pass^k`，它衡量的是：对于某一任务，独立运行 `k` 次时“这 k 次全部都成功”的概率，再对所有任务求平均。它不同于其他 LLM 评测中常见的 `pass@k`。`Pass@k` 衡量的是“k 次尝试里至少有一次成功”，而 `pass^k` 衡量的是一致性和可靠性，这对于客服应用尤其关键，因为客服系统必须稳定地遵守策略。

### 表现分析

我们的评估比较了几种不同配置：

- 基线：没有 `think` 工具，也没有 `extended thinking`
- 仅 `extended thinking`
- 仅 `think` 工具
- `think` 工具 + 优化过的 prompt（针对航空场景）

结果显示，当 Claude 3.7 正确使用 `think` 工具时，无论是在 `τ-Bench` 的“航空”还是“零售”客服场景中，都带来了明显改进。

- 航空场景：带优化 prompt 的 `think` 工具在 `pass^1` 指标上达到 0.570，而基线只有 0.370，相对提升达 54%
- 零售场景：仅 `think` 工具就达到 0.812，而基线为 0.783

Claude 3.7 Sonnet 在 `Tau-Bench` 的航空场景中的配置结果如下：

| 配置 | k=1 | k=2 | k=3 | k=4 | k=5 |
|---|---:|---:|---:|---:|---:|
| `think` + prompt | 0.584 | 0.444 | 0.384 | 0.356 | 0.340 |
| `think` | 0.404 | 0.254 | 0.186 | 0.140 | 0.100 |
| `extended thinking` | 0.412 | 0.290 | 0.232 | 0.192 | 0.160 |
| 基线 | 0.332 | 0.206 | 0.148 | 0.116 | 0.100 |

在航空场景中，最佳结果来自“`think` 工具 + 优化 prompt”的组合。这个 prompt 会给出示例，说明在分析客户请求时应该采用什么样的推理方式。下面是一个优化 prompt 示例：

```text
## Using the think tool

Before taking any action or responding to the user after receiving tool results, use the think tool as a scratchpad to:
- List the specific rules that apply to the current request
- Check if all required information is collected
- Verify that the planned action complies with all policies
- Iterate over tool results for correctness

Here are some examples of what to iterate over inside the think tool:
<think_tool_example_1>
User wants to cancel flight ABC123
- Need to verify: user ID, reservation ID, reason
- Check cancellation rules:
  * Is it within 24h of booking?
  * If not, check ticket class and insurance
- Verify no segments flown or are in the past
- Plan: collect missing info, verify rules, get confirmation
</think_tool_example_1>

<think_tool_example_2>
User wants to book 3 tickets to NYC with 2 checked bags each
- Need user ID to check:
  * Membership tier for baggage allowance
  * Which payments methods exist in profile
- Baggage calculation:
  * Economy class × 3 passengers
  * If regular member: 1 free bag each → 3 extra bags = $150
  * If silver member: 2 free bags each → 0 extra bags = $0
  * If gold member: 3 free bags each → 0 extra bags = $0
- Payment rules to verify:
  * Max 1 travel certificate, 1 credit card, 3 gift cards
  * All payment methods must be in profile
  * Travel certificate remainder goes to waste
- Plan:
1. Get user ID
2. Verify membership level for bag fees
3. Check which payment methods in profile and if their combination is allowed
4. Calculate total: ticket price + any bag fees
5. Get explicit confirmation for booking
</think_tool_example_2>
```

这里一个特别有意思的点是，不同方法之间的差异相当明显。使用带优化 prompt 的 `think` 工具，其表现显著优于 `extended thinking` 模式，而 `extended thinking` 的表现与“未加提示词的 `think` 工具”大致接近。单独使用 `think` 工具，虽然比基线更好，但仍然不如优化后的方案。

`Think` 工具与优化 prompt 的组合之所以显著领先，很可能是因为航空场景中的策略规则非常复杂，模型在这种情况下尤其受益于“应该如何思考”的具体示例。

在零售场景中，我们也测试了多种配置，以理解每种方法的具体影响。

Claude 3.7 Sonnet 在 `Tau-Bench` 零售场景中的表现如下：

| 配置 | k=1 | k=2 | k=3 | k=4 | k=5 |
|---|---:|---:|---:|---:|---:|
| `think` + 无 prompt | 0.812 | 0.735 | 0.685 | 0.650 | 0.626 |
| `extended thinking` | 0.770 | 0.681 | 0.623 | 0.581 | 0.548 |
| 基线 | 0.783 | 0.695 | 0.643 | 0.607 | 0.583 |

即使没有额外 prompt，`think` 工具也达到了最高的 `pass^1` 分数 0.812。相比航空场景，零售策略更容易处理，因此 Claude 仅仅因为有了一个“可以思考的空间”，表现就已经能提升，而不需要额外指导。

### 从 τ-Bench 分析中得到的关键洞见

我们的详细分析发现了一些模式，可以帮助你更有效地实现 `think` 工具：

- 在困难场景中，prompt 非常重要。仅仅提供 `think` 工具，可能会带来一些提升；但如果再配合优化后的提示词，困难场景中的收益会显著更大。不过在较简单场景中，光有 `think` 工具也可能已经有帮助。
- 跨多次试验的一致性提升。使用 `think` 后，`pass^k` 在直到 `k=5` 的范围内都保持了改进，这说明这个工具帮助 Claude 更好地处理边缘案例和不常见情形。

## 在 SWE-Bench 上的表现

在评估 Claude 3.7 Sonnet 时，我们也在 `SWE-bench` 的设置中加入了一个类似的 `think` 工具，而这也是它取得 0.623 这一当时最先进分数的一部分原因。适配后的 `think` 工具定义如下：

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or make any changes to the repository, but just log the thought. Use it when complex reasoning or brainstorming is needed. For example, if you explore the repo and discover the source of a bug, call this tool to brainstorm several unique ways of fixing the bug, and assess which change(s) are likely to be simplest and most effective. Alternatively, if you receive some test results, call this tool to brainstorm ways to fix the failing tests.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "Your thoughts."
      }
    },
    "required": ["thought"]
  }
}
```

我们的实验结果显示，在只隔离考察“是否加入该工具”的条件下，加入 `think` 工具平均可提升 1.6% 的表现（`n=30` 个带 `think` 工具样本，`n=144` 个不带样本；Welch's t-test: `t(38.89) = 6.71, p < .001, d = 1.47`）。

## 什么时候适合使用 “think” 工具

基于这些评测结果，我们总结出 Claude 最能从 `think` 工具受益的几类场景：

- `工具输出分析`：Claude 在采取行动之前，需要仔细处理先前工具调用的结果，并且可能需要回退和调整策略。
- `策略密集型环境`：Claude 需要遵守详细规则，并验证自己的操作是否合规。
- `顺序型决策`：每一步都会建立在前一步基础上，错误代价高，常见于多步骤领域。

## 实现最佳实践

为了让 Claude 最大程度地从 `think` 工具中受益，我们基于 `τ-Bench` 实验，建议采用以下实践。

### 1. 结合特定领域示例做策略性 prompting

最有效的方法，是像 `τ-Bench` 的航空场景那样，明确告诉 Claude 什么时候该使用 `think` 工具，以及该如何使用它。提供与你具体用例相关的示例，可以显著提升模型使用 `think` 工具的效果，例如：

- 推理过程应该达到什么样的细致程度
- 如何把复杂指令拆解成可执行步骤
- 如何为常见场景构建决策树
- 如何检查是否已经收集到所有必要信息

### 2. 把复杂指导放在 system prompt 中

我们发现，当关于 `think` 工具的说明较长或较复杂时，把这些说明放在 `system prompt` 中，比放进工具描述本身更有效。这样做能提供更广泛的上下文，也能帮助模型把思考过程更自然地整合进整体行为模式中。

## 什么时候不适合使用 “think” 工具

虽然 `think` 工具在正确场景下能带来明显提升，但它并不适用于所有工具使用场景，而且会增加 prompt 长度与输出 token 成本。我们发现，在以下场景中，`think` 工具并没有带来改善：

- `非顺序型工具调用`：如果 Claude 只需进行一次工具调用，或者几次并行调用就能完成任务，那么加入 `think` 通常不会带来改进。
- `简单的指令遵循`：如果 Claude 不需要满足太多约束，而且默认行为已经足够好，那么额外“思考”往往不会带来收益。

## 开始上手

把 `think` 工具加入 Claude 的实现非常直接，而且只需几步就可能带来有意义的提升：

- 在 agentic 工具使用场景中测试它。优先选择那些 Claude 当前在策略遵循、长工具链复杂推理方面表现较差的任务。
- 加入工具定义。实现一个适合你领域的 `think` 工具。它所需代码极少，却能让推理变得更结构化。同时也建议在 `system prompt` 中加入关于何时以及如何使用这个工具的说明，并给出与你领域相关的示例。
- 监控并持续优化。观察 Claude 在实际中如何使用这个工具，并调整 prompt，引导它形成更高效的思考模式。

这件事最大的好处在于，它在性能层面几乎没有明显副作用。只有在 Claude 主动决定使用时，这个工具才会改变行为，而且它不会干扰你现有的工具或工作流。

## 结论

我们的研究表明，`think` 工具可以显著提升 Claude 3.7 Sonnet<sup>1</sup> 在复杂任务中的表现，尤其是在需要遵循策略以及在长工具链中进行推理的场景下。`Think` 不是一个适用于所有问题的万能方案，但在合适用例中，它能以极低的实现复杂度带来相当可观的收益。

我们很期待看到你如何使用 `think` 工具，基于 Claude 构建出更强大、更可靠、更透明的 AI 系统。

<sup>1</sup> 虽然我们的 `τ-Bench` 结果重点展示了 `think` 工具对 Claude 3.7 Sonnet 的提升，但实验也表明，Claude 3.5 Sonnet（新版）在与 3.7 Sonnet 相同配置下同样可以获得性能提升，这说明这种改进对其他 Claude 模型也具有泛化性。
