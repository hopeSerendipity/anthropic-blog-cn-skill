# Introducing advanced tool use on the Claude Developer Platform - 中文翻译

发布时间：`2025-11-24`

我们新增了三个测试版功能，让 Claude 能够动态发现、学习并执行工具。下面介绍它们的工作方式。

AI agent 的未来，是模型能够在数百甚至数千种工具之间无缝协作。比如，一个 IDE 助手可以集成 git 操作、文件处理、包管理器、测试框架和部署流水线；一个运维协调 agent 可以同时连接 Slack、GitHub、Google Drive、Jira、公司数据库以及大量 MCP server。

如果要构建高效的 agent，它们就必须能够使用近乎无限的工具库，而不是在一开始就把所有工具定义全部塞进上下文里。我们之前关于通过 MCP 使用代码执行的博客文章曾讨论过，工具结果和工具定义有时会在 agent 读到用户请求之前，就先消耗掉 50,000 多个 tokens。理想的 agent 应该按需发现和加载工具，只保留当前任务真正相关的部分。

Agent 还需要能够在代码中调用工具。使用自然语言方式调用工具时，每次调用都需要完整跑一次推理，而中间结果会不断堆进上下文，不管这些结果是否真的有用。代码天然适合处理编排逻辑，例如循环、条件判断和数据转换。因此，agent 需要能够根据任务本身，在“代码执行”和“模型推理”之间灵活选择。

Agent 还需要从示例中学习如何正确使用工具，而不只是看 schema 定义。JSON schema 只能定义“结构上是否合法”，却无法表达“实际应该怎么用”，例如什么时候要传可选参数、哪些参数组合才合理、你的 API 默认遵循什么约定。

今天，我们发布了三个让这些能力成为可能的新特性：

- `Tool Search Tool`：允许 Claude 使用搜索工具访问数千个工具，而不占用大量上下文窗口。
- `Programmatic Tool Calling`：允许 Claude 在代码执行环境中调用工具，从而降低模型上下文窗口的压力。
- `Tool Use Examples`：提供一种通用标准，用于展示某个工具应该如何被高效使用。

在内部测试中，我们发现这些能力帮助我们构建了一些用传统工具调用模式根本做不到的系统。例如，Claude for Excel 就借助 `Programmatic Tool Calling` 来读取和修改包含数千行数据的电子表格，同时不会压垮模型的上下文窗口。

根据我们的经验，我们相信这些能力为使用 Claude 构建系统打开了新的空间。

## Tool Search Tool

### 挑战

MCP 工具定义会提供重要上下文，但随着接入的 server 越来越多，这些 tokens 会快速累积。比如，一个包含五个 server 的环境：

- GitHub：35 个工具，大约 26K tokens
- Slack：11 个工具，大约 21K tokens
- Sentry：5 个工具，大约 3K tokens
- Grafana：5 个工具，大约 3K tokens
- Splunk：2 个工具，大约 2K tokens

这意味着，在对话还没开始之前，仅 58 个工具定义就已经消耗了大约 55K tokens。再加上一些额外 server，比如 Jira 单独就大约占 17K tokens，很快就会逼近 100K+ 的开销。在 Anthropic 内部，我们甚至见过工具定义在优化前先吃掉 134K tokens。

但问题不只是 token 成本。最常见的失败模式，是选错工具或者传错参数，尤其是当工具名很接近时，例如 `notification-send-user` 和 `notification-send-channel`。

### 我们的方案

与其一开始就把所有工具定义都加载进上下文，不如让 `Tool Search Tool` 按需发现工具。这样，Claude 只会看到当前任务真正需要的工具。

传统方式：

- 一开始就加载全部工具定义，例如 50+ 个 MCP 工具就可能占用约 72K tokens
- 对话历史和 system prompt 只能去争抢剩余空间
- 在真正开始工作之前，总上下文消耗可能就达到约 77K tokens

使用 `Tool Search Tool` 时：

- 一开始只加载 `Tool Search Tool` 自己，大约只占 500 tokens
- 需要时再按需发现工具，通常只加载 3 到 5 个相关工具，大约 3K tokens
- 总上下文消耗大约只需 8.7K tokens，保留了约 95% 的上下文窗口

这意味着，在仍然可以访问完整工具库的前提下，token 使用量减少了 85%。内部测试还显示，在大型工具库场景下，启用 `Tool Search Tool` 能明显提升 MCP 评测准确率。`Opus 4` 从 49% 提升到 74%，`Opus 4.5` 从 79.5% 提升到 88.1%。

### Tool Search Tool 的工作方式

`Tool Search Tool` 允许 Claude 动态发现工具，而不是提前把所有工具定义都加载进来。你需要把所有工具定义仍然传给 API，但通过 `defer_loading: true` 把某些工具标记为“按需加载”。这类延迟加载工具不会在最初进入 Claude 的上下文。Claude 一开始只看到 `Tool Search Tool` 本身，以及那些设置了 `defer_loading: false` 的关键高频工具。

当 Claude 需要某种能力时，它会搜索相关工具。`Tool Search Tool` 会返回与之匹配的工具引用，这些引用随后被展开成完整工具定义并加载进 Claude 的上下文。

例如，如果 Claude 需要与 GitHub 交互，它会搜索 “github”，然后只加载 `github.createPullRequest` 和 `github.listIssues`，而不会把来自 Slack、Jira、Google Drive 的其他 50 多个工具一起加载进来。

因此，Claude 可以访问你的完整工具库，但只为真正需要的工具支付 token 成本。

关于 prompt caching：`Tool Search Tool` 不会破坏 prompt caching，因为延迟加载工具根本不会出现在初始 prompt 里。它们只有在 Claude 触发搜索之后才会进入上下文，因此 system prompt 和核心工具定义仍然是可缓存的。

实现方式如下：

```json
{
  "tools": [
    {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
    {
      "name": "github.createPullRequest",
      "description": "Create a pull request",
      "input_schema": {...},
      "defer_loading": true
    }
  ]
}
```

对于 MCP server，你也可以把整个 server 延迟加载，但保留少数高频工具始终常驻：

```json
{
  "type": "mcp_toolset",
  "mcp_server_name": "google-drive",
  "default_config": {"defer_loading": true},
  "configs": {
    "search_files": {
      "defer_loading": false
    }
  }
}
```

Claude Developer Platform 已经内置了基于正则和基于 BM25 的搜索工具，但你也可以用 embedding 或其他策略自己实现自定义搜索工具。

### 什么时候适合使用 Tool Search Tool

和任何架构决策一样，启用 `Tool Search Tool` 也存在取舍。因为工具调用前要多一个搜索步骤，所以只有当节省的上下文和提升的准确率大于增加的延迟时，收益才最明显。

适合使用的场景：

- 工具定义总共占用超过 10K tokens
- 已经出现工具选择准确率问题
- 正在构建由多个 MCP server 驱动的系统
- 可用工具数量达到 10 个以上

不太适合的场景：

- 工具库很小，小于 10 个工具
- 每次会话几乎都会频繁用到所有工具
- 工具定义本身就很紧凑

## Programmatic Tool Calling

### 挑战

随着工作流越来越复杂，传统工具调用会暴露两个根本问题。

第一是中间结果污染上下文。比如，Claude 在 10MB 的日志文件里找错误模式时，整份日志都会进入它的上下文，哪怕它真正需要的可能只是错误频率的汇总。又比如，在多个表之间查询客户数据时，不管记录是否相关，所有记录都会不断累积进上下文。这些中间结果会大量消耗 token 预算，甚至把真正重要的信息挤出上下文窗口。

第二是推理开销和手工综合成本。每一次工具调用都需要完整跑一轮模型推理。拿到结果后，Claude 还必须用自然语言去“目测”数据、抽取相关信息、判断不同片段之间如何关联，并决定下一步操作。一个五步工具工作流就意味着五轮模型推理，以及 Claude 对每一步结果的重新解析、对比和综合，这既慢又容易出错。

### 我们的方案

`Programmatic Tool Calling` 允许 Claude 通过代码来编排工具，而不再通过一次次独立的 API 往返来调用工具。也就是说，Claude 不再是一轮一轮请求工具、再把每一步结果都送回自己上下文，而是会写一段代码，在其中调用多个工具、处理工具输出，并自己决定最终有哪些信息真的需要进入上下文窗口。

Claude 很擅长写代码。让它用 Python 来表达编排逻辑，而不是用自然语言反复调用工具，能带来更可靠、更精确的控制流。循环、条件判断、数据转换和错误处理，都会在代码里显式写出来，而不是隐含在 Claude 的自然语言推理中。

### 示例：预算合规检查

假设你有一个常见业务任务：“哪些团队成员超出了 Q3 差旅预算？”

你有三种工具：

- `get_team_members(department)`：返回团队成员列表，包括 ID 和职级
- `get_expenses(user_id, quarter)`：返回某个员工某季度的费用明细
- `get_budget_by_level(level)`：返回某一员工职级对应的预算上限

传统方式下：

- 先获取团队成员，例如 20 人
- 对每个人获取其 Q3 开销，意味着 20 次工具调用，每次返回 50 到 100 条费用明细，比如机票、酒店、餐饮、报销单
- 再按员工职级获取预算上限
- 所有这些内容都会进入 Claude 的上下文：2000+ 条费用明细，超过 50KB
- Claude 还要手工把每个人的花费加总，再查预算，再做比对
- 这会带来更多模型往返以及显著的上下文消耗

使用 `Programmatic Tool Calling` 时：

Claude 不会把每一步结果都拉回自己上下文，而是先写一段 Python 脚本，让脚本完成整个编排流程。脚本运行在 `Code Execution` 工具的沙箱环境中，只有在需要某个工具结果时才暂停。你通过 API 把工具结果返回后，这些结果会先被脚本消费，而不是被模型消费。脚本继续运行，直到最后只把最终结果返回给 Claude。

用于预算检查任务的编排代码示例如下：

```python
team = await get_team_members("engineering")

levels = list(set(m["level"] for m in team))
budget_results = await asyncio.gather(*[
    get_budget_by_level(level) for level in levels
])

budgets = {level: budget for level, budget in zip(levels, budget_results)}

expenses = await asyncio.gather(*[
    get_expenses(m["id"], "Q3") for m in team
])

exceeded = []
for member, exp in zip(team, expenses):
    budget = budgets[member["level"]]
    total = sum(e["amount"] for e in exp)
    if total > budget["travel_limit"]:
        exceeded.append({
            "name": member["name"],
            "spent": total,
            "limit": budget["travel_limit"]
        })

print(json.dumps(exceeded))
```

Claude 最终只会看到最终输出，也就是那两三个超预算的人。那 2000 多条费用明细、中间求和结果和预算查找过程，都不会进入 Claude 的上下文。这会把上下文消耗从 200KB 的原始费用数据压缩到大约 1KB 的结果。

这种方式带来的收益很明显：

- `Token 节省`：中间结果不进入 Claude 上下文，复杂研究任务上的平均 token 使用量从 43,588 降到 27,297，下降了 37%。
- `延迟下降`：每次 API 往返都要触发模型推理，可能是几百毫秒到几秒。如果 Claude 在一段代码中编排 20 次以上工具调用，就可以省掉 19 次以上的推理往返。
- `准确率提升`：显式代码编排比在自然语言里 juggling 多个工具结果更不容易出错。内部知识检索从 25.6% 提升到 28.5%，GIA 基准从 46.5% 提升到 51.2%。

现实生产工作流往往充满脏数据、条件逻辑和可扩展性需求。`Programmatic Tool Calling` 让 Claude 以编程方式处理这些复杂性，同时把自身注意力集中在“可执行结果”而不是原始数据加工上。

### Programmatic Tool Calling 的工作方式

#### 1. 把工具标记为可从代码调用

要把 `code_execution` 加入工具列表，并用 `allowed_callers` 显式选择允许被代码执行环境调用的工具：

```json
{
  "tools": [
    {
      "type": "code_execution_20250825",
      "name": "code_execution"
    },
    {
      "name": "get_team_members",
      "description": "Get all members of a department...",
      "input_schema": {...},
      "allowed_callers": ["code_execution_20250825"]
    }
  ]
}
```

API 会把这些工具定义转换成 Claude 可调用的 Python 函数。

#### 2. Claude 编写编排代码

Claude 不再一次次请求单独工具，而是生成 Python 代码：

```json
{
  "type": "server_tool_use",
  "id": "srvtoolu_abc",
  "name": "code_execution",
  "input": {
    "code": "team = get_team_members('engineering')\n..."
  }
}
```

#### 3. 工具执行不会污染 Claude 的上下文

当代码调用 `get_expenses()` 时，你会收到一个带有 `caller` 字段的工具请求：

```json
{
  "type": "tool_use",
  "id": "toolu_xyz",
  "name": "get_expenses",
  "input": {"user_id": "emp_123", "quarter": "Q3"},
  "caller": {
    "type": "code_execution_20250825",
    "tool_id": "srvtoolu_abc"
  }
}
```

你把结果返回后，这些结果是在 `Code Execution` 环境中被脚本处理，而不是送回 Claude 上下文。这个请求-响应过程会随着代码里的每次工具调用持续进行。

#### 4. 只有最终输出才进入上下文

当代码运行结束后，只有代码的最终输出会返回给 Claude：

```json
{
  "type": "code_execution_tool_result",
  "tool_use_id": "srvtoolu_abc",
  "content": {
    "stdout": "[{\"name\": \"Alice\", \"spent\": 12500, \"limit\": 10000}...]"
  }
}
```

Claude 看到的只有这些结果，而不是过程中处理过的 2000 多条费用明细。

### 什么时候适合使用 Programmatic Tool Calling

`Programmatic Tool Calling` 会在工作流中额外增加一个代码执行步骤。只有当 token 节省、延迟收益和准确率提升足够明显时，这个开销才是值得的。

最适合的场景：

- 需要处理大数据集，但最终只需要聚合值或摘要
- 有三个及以上相互依赖的多步工具调用
- 在 Claude 看见结果之前，需要先过滤、排序或转换工具输出
- 中间数据不应该干扰 Claude 推理
- 需要对大量对象执行并行操作，例如检查 50 个 endpoint

不太适合的场景：

- 只是简单的一次单工具调用
- Claude 本来就应该看见并推理所有中间结果
- 只是做小型、快速的查找

## Tool Use Examples

### 挑战

JSON Schema 很擅长定义结构，例如类型、必填字段、允许的枚举值，但它没法表达“使用模式”，例如什么时候该加可选参数、哪些参数组合合理、你的 API 实际遵循什么约定。

例如，一个支持工单系统的 API：

```json
{
  "name": "create_ticket",
  "input_schema": {
    "properties": {
      "title": {"type": "string"},
      "priority": {"enum": ["low", "medium", "high", "critical"]},
      "labels": {"type": "array", "items": {"type": "string"}},
      "reporter": {
        "type": "object",
        "properties": {
          "id": {"type": "string"},
          "name": {"type": "string"},
          "contact": {
            "type": "object",
            "properties": {
              "email": {"type": "string"},
              "phone": {"type": "string"}
            }
          }
        }
      },
      "due_date": {"type": "string"},
      "escalation": {
        "type": "object",
        "properties": {
          "level": {"type": "integer"},
          "notify_manager": {"type": "boolean"},
          "sla_hours": {"type": "integer"}
        }
      }
    },
    "required": ["title"]
  }
}
```

Schema 确实定义了什么叫“合法”，但仍然留下了很多关键问题：

- `格式歧义`：`due_date` 应该写成 `"2024-11-06"`、`"Nov 6, 2024"`，还是 `"2024-11-06T00:00:00Z"`？
- `ID 约定`：`reporter.id` 是 UUID、`"USR-12345"`，还是纯 `"12345"`？
- `嵌套结构使用方式`：Claude 应该在什么情况下填充 `reporter.contact`？
- `参数关联`：`escalation.level` 和 `escalation.sla_hours` 与 `priority` 之间是什么关系？

这些歧义都会导致工具调用格式错误，或者参数使用不一致。

### 我们的方案

`Tool Use Examples` 允许你直接在工具定义里提供样例调用。这样一来，Claude 不再只是依赖 schema，而是能够从具体示例中学习正确用法：

```json
{
  "name": "create_ticket",
  "input_schema": { /* same schema as above */ },
  "input_examples": [
    {
      "title": "Login page returns 500 error",
      "priority": "critical",
      "labels": ["bug", "authentication", "production"],
      "reporter": {
        "id": "USR-12345",
        "name": "Jane Smith",
        "contact": {
          "email": "jane@acme.com",
          "phone": "+1-555-0123"
        }
      },
      "due_date": "2024-11-06",
      "escalation": {
        "level": 2,
        "notify_manager": true,
        "sla_hours": 4
      }
    },
    {
      "title": "Add dark mode support",
      "labels": ["feature-request", "ui"],
      "reporter": {
        "id": "USR-67890",
        "name": "Alex Chen"
      }
    },
    {
      "title": "Update API documentation"
    }
  ]
}
```

Claude 可以从这三个例子里学到：

- `格式约定`：日期使用 `YYYY-MM-DD`，用户 ID 遵循 `USR-XXXXX`，标签使用 `kebab-case`
- `嵌套结构模式`：如何正确构造 `reporter` 以及嵌套的 `contact`
- `可选参数之间的关联`：严重 bug 通常带完整联系方式和带严格 SLA 的升级信息；功能请求可能有 reporter 但没有 contact 或 escalation；内部任务可能只需要 title

在我们的内部测试中，`Tool Use Examples` 让复杂参数处理任务的准确率从 72% 提升到 90%。

### 什么时候适合使用 Tool Use Examples

`Tool Use Examples` 会增加工具定义里的 token，因此只有在准确率收益大于额外成本时，它才最有价值。

最适合的场景：

- 结构复杂、嵌套较深，合法 JSON 不等于正确用法
- 工具有很多可选参数，而且参数是否出现本身很重要
- API 有 schema 表达不出来的领域约定
- 工具之间很相似，而样例有助于区分它们，例如 `create_ticket` 和 `create_incident`

不太适合的场景：

- 只有一个简单参数、而且用法一眼就懂的工具
- Claude 本来就熟悉的标准格式，比如 URL 或邮箱
- 更适合通过 JSON Schema 约束解决的校验问题

## 最佳实践

构建能在真实世界中采取行动的 agent，意味着你必须同时处理规模、复杂性和精度。这三个新特性分别解决工具调用流程中的不同瓶颈，而且它们能够彼此配合。下面是更有效的组合方式。

### 按瓶颈分层使用这些能力

并不是每个 agent 在每个任务里都需要同时启用这三项能力。最好的方式，是先从你当前最大的瓶颈开始：

- 工具定义造成上下文膨胀 -> `Tool Search Tool`
- 大量中间结果污染上下文 -> `Programmatic Tool Calling`
- 参数错误和工具调用格式错误 -> `Tool Use Examples`

这种方式能帮助你优先解决真正限制 agent 表现的关键约束，而不是一上来就把复杂度全部加满。

然后，再根据需要叠加其他能力。它们之间是互补的：`Tool Search Tool` 保证找到正确工具，`Programmatic Tool Calling` 保证执行高效，`Tool Use Examples` 保证调用正确。

### 设置 Tool Search Tool 以提升发现质量

工具搜索是基于名称和描述匹配的，因此清晰且有描述性的定义，会直接提升发现准确率。

好的定义：

```json
{
  "name": "search_customer_orders",
  "description": "Search for customer orders by date range, status, or total amount. Returns order details including items, shipping, and payment info."
}
```

糟糕的定义：

```json
{
  "name": "query_db_orders",
  "description": "Execute order query"
}
```

你还应该在 system prompt 里补充引导，让 Claude 知道当前有哪些能力域可用：

```text
You have access to tools for Slack messaging, Google Drive file management,
Jira ticket tracking, and GitHub repository operations. Use the tool search
to find specific capabilities.
```

建议把最常用的 3 到 5 个工具始终保持常驻，其余工具按需延迟加载。这样，既能保证常用操作的即时性，也能兼顾整套工具库的按需发现。

### 设置 Programmatic Tool Calling 以确保执行正确

因为 Claude 会写代码去解析工具输出，所以你需要把返回格式说明写得足够清楚，这样它才能写出正确的解析逻辑：

```json
{
  "name": "get_orders",
  "description": "Retrieve orders for a customer.\nReturns:\n    List of order objects, each containing:\n    - id (str): Order identifier\n    - total (float): Order total in USD\n    - status (str): One of 'pending', 'shipped', 'delivered'\n    - items (list): Array of {sku, quantity, price}\n    - created_at (str): ISO 8601 timestamp"
}
```

特别适合加入 `Programmatic Tool Calling` 的工具包括：

- 可以并行执行的独立操作
- 安全可重试的幂等操作

### 设置 Tool Use Examples 以提高参数准确率

示例应当服务于行为清晰性：

- 使用真实感数据，例如真实城市名、合理价格，而不是 `"string"` 或 `"value"`
- 同时展示最小、部分和完整三种使用模式
- 保持简洁，每个工具给 1 到 5 个示例即可
- 聚焦歧义点，只在 schema 无法清楚表达正确用法时才补示例

## 开始使用

这些能力目前以 beta 形式提供。启用方式是添加 beta header，并在请求中加入所需工具：

```python
client.beta.messages.create(
    betas=["advanced-tool-use-2025-11-20"],
    model="claude-sonnet-4-5-20250929",
    max_tokens=4096,
    tools=[
        {"type": "tool_search_tool_regex_20251119", "name": "tool_search_tool_regex"},
        {"type": "code_execution_20250825", "name": "code_execution"},
        # Your tools with defer_loading, allowed_callers, and input_examples
    ]
)
```

更详细的 API 文档和 SDK 示例可参考：

- `Tool Search Tool` 的文档与 cookbook
- `Programmatic Tool Calling` 的文档与 cookbook
- `Tool Use Examples` 的文档

这些能力让工具使用从简单的 function calling，进一步走向更智能的编排。当 agent 开始处理跨越数十个工具和大规模数据集的复杂工作流时，动态发现、高效执行以及可靠调用，将成为基础能力。

我们很期待看到你们会构建出什么。

## 致谢

本文由 Bin Wu 撰写，Adam Jones、Artur Renault、Henry Tay、Jake Noble、Noah Picard、Sam Jiang 以及 Claude Developer Platform 团队参与贡献。这项工作建立在 Chris Gorgolewski、Daniel Jiang、Jeremy Fox 和 Mike Lambert 的基础研究之上。我们也从更广泛的 AI 生态中获得灵感，包括 Joel Pobar 的 LLMVM、Cloudflare 的 Code Mode，以及 Code Execution as MCP。特别感谢 Andy Schumeister、Hamish Kerr、Keir Bradwell、Matt Bleifer 和 Molly Vorwerck 提供支持。
