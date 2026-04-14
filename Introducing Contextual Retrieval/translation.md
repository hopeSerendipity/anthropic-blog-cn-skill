# Introducing Contextual Retrieval - 中文翻译

发布时间：`2024-09-19`

为了让 AI 模型在特定上下文中真正有用，它通常需要能够访问背景知识。

例如，客服聊天机器人需要了解其所服务的具体业务，而法律分析机器人则需要了解大量过往案例。

开发者通常会通过 `Retrieval-Augmented Generation`（`RAG`，检索增强生成）来增强 AI 模型的知识。RAG 的做法是从知识库中检索相关信息，并把这些信息附加到用户 prompt 中，从而显著提升模型的回答质量。问题在于，传统 RAG 方案在编码信息时会移除上下文，而这往往会导致系统无法从知识库中检索到真正相关的信息。

在这篇文章中，我们介绍一种能显著改进 RAG 检索步骤的方法。这种方法名为 `Contextual Retrieval`，它由两个子技术构成：`Contextual Embeddings` 和 `Contextual BM25`。这种方法可以将检索失败次数减少 49%；如果再与 reranking 结合，失败次数可减少 67%。这意味着检索准确率出现了显著提升，而这会直接转化为下游任务表现的改善。

你可以通过我们的 cookbook，轻松用 Claude 部署自己的 `Contextual Retrieval` 方案。

## 关于“直接用更长 prompt”的一个说明

有时候，最简单的办法反而是最好的。如果你的知识库小于 200,000 tokens，大约相当于 500 页材料，那么你完全可以直接把整个知识库都放进发送给模型的 prompt 中，而不需要使用 RAG 或类似方法。

几周前，我们发布了 Claude 的 `prompt caching`，这让这种做法在速度和成本上都变得更可行。开发者现在可以在多次 API 调用之间缓存高频使用的 prompt，从而将延迟降低超过 2 倍，并将成本最多降低 90%。

不过，当你的知识库继续增长时，你仍然需要一个更具扩展性的方案。这就是 `Contextual Retrieval` 的用武之地。

## RAG 入门：扩展到更大的知识库

对于无法放进上下文窗口的大型知识库，RAG 通常是标准解决方案。RAG 的预处理步骤通常如下：

- 将知识库，也就是文档语料，切分成更小的文本块，通常每块不超过几百个 tokens；
- 使用 embedding 模型把这些文本块转换成能够编码语义的向量表示；
- 将这些向量存储进向量数据库，以支持基于语义相似度的搜索。

在运行时，当用户提交查询后，系统会用向量数据库根据语义相似度找到最相关的文本块，然后把这些文本块加入发送给生成模型的 prompt 中。

虽然 embedding 模型很擅长捕捉语义关系，但它们有时会漏掉关键的精确匹配。幸运的是，有一种更古老的技术可以在这类场景中提供帮助，那就是 `BM25`（Best Matching 25）。BM25 是一种基于词汇匹配的排序函数，擅长查找精确的单词或短语匹配。对于包含唯一标识符或技术术语的查询，它尤其有效。

BM25 建立在 `TF-IDF`（词频-逆文档频率）概念之上。TF-IDF 用来衡量一个词在整个文档集合中对某篇文档的重要性，而 BM25 在此基础上进一步考虑了文档长度，并对词频应用了饱和函数，以避免高频常见词主导结果。

下面是 BM25 在 embedding 失败时如何成功的一个例子：假设用户在一个技术支持数据库里查询 “Error code TS-999”。Embedding 模型可能会找到许多和“错误码”相关的一般性内容，却漏掉精确的 “TS-999” 匹配；而 BM25 会专门去查找这一精确文本，从而定位到相关文档。

通过结合 embeddings 和 BM25，RAG 系统可以更准确地检索到最适用的文本块。具体流程如下：

- 将知识库切成小块，通常每块不超过几百个 tokens；
- 为这些文本块生成 TF-IDF 编码和语义 embeddings；
- 使用 BM25 找到基于精确匹配的高分文本块；
- 使用 embeddings 找到基于语义相似度的高分文本块；
- 使用 rank fusion 技术把两组结果合并并去重；
- 将 top-K 文本块加入 prompt，生成最终响应。

借助 BM25 与 embedding 模型，传统 RAG 系统就能在精确术语匹配和广义语义理解之间取得平衡，从而得到更全面、更准确的检索结果。

这一方案能让你以较低成本扩展到极大的知识库，远超单次 prompt 可承载的规模。但传统 RAG 仍有一个重大限制：它常常会破坏上下文。

## 传统 RAG 中的上下文困境

在传统 RAG 里，文档通常会被切成更小的块，以便高效检索。虽然这种方法在很多应用中表现良好，但当某个单独的 chunk 本身缺乏足够上下文时，就会出现问题。

例如，假设你的知识库里包含一批财务信息，比如美国 SEC 文件，而你收到的问题是：“ACME Corp 在 2023 年第二季度的收入增长是多少？”

此时，一个相关 chunk 可能只包含这样一句话：“该公司的收入较上一季度增长了 3%。”然而，这个 chunk 单独看并没有说明它指的是哪家公司，也没有说明对应的是哪个时间段，这就使得系统很难正确检索到这条信息，或者在检索到了之后有效使用它。

## 引入 Contextual Retrieval

`Contextual Retrieval` 就是为了解决这个问题。它的核心做法是：在对每个 chunk 做 embedding 之前，先在 chunk 前面加上一段只属于该 chunk 的解释性上下文，这部分称为 `Contextual Embeddings`；同时，在构建 BM25 索引时，也使用这份带上下文的 chunk，这部分称为 `Contextual BM25`。

回到刚才的 SEC 文件示例，chunk 的变化大致如下：

```python
original_chunk = "The company's revenue grew by 3% over the previous quarter."

contextualized_chunk = "This chunk is from an SEC filing on ACME corp's performance in Q2 2023; the previous quarter's revenue was $314 million. The company's revenue grew by 3% over the previous quarter."
```

值得一提的是，过去也有人提出过利用上下文提升检索表现的方法。例如：给 chunk 添加通用文档摘要、使用 hypothetical document embedding，以及基于摘要建立索引等。我们也做过实验，但看到的收益非常有限，甚至表现较低。本文提出的方法与这些方案不同。

## 如何实现 Contextual Retrieval

当然，如果知识库中有成千上万甚至数百万个 chunk，靠人工给每个 chunk 写注释几乎是不可能的。因此，在实现 `Contextual Retrieval` 时，我们选择使用 Claude。我们编写了一个 prompt，要求模型根据整个文档的上下文，为每个 chunk 生成一段简短、只针对该 chunk 的上下文说明。我们使用了如下 Claude 3 Haiku prompt：

```text
<document>
{{WHOLE_DOCUMENT}}
</document>
Here is the chunk we want to situate within the whole document
<chunk>
{{CHUNK_CONTENT}}
</chunk>
Please give a short succinct context to situate this chunk within the overall document for the purposes of improving search retrieval of the chunk. Answer only with the succinct context and nothing else.
```

生成出来的 contextual text 通常是 50 到 100 个 tokens。我们会把它拼接到原始 chunk 前面，然后再用这个“带上下文的 chunk”去做 embedding 和构建 BM25 索引。

实践中的预处理流程大致就是如此。`Contextual Retrieval` 本质上是一种预处理技术，用于提升检索准确率。

如果你想尝试 `Contextual Retrieval`，可以直接从我们的 cookbook 开始。

## 使用 Prompt Caching 来降低 Contextual Retrieval 成本

`Contextual Retrieval` 之所以能以较低成本落地，很大程度上依赖于 Claude 的 `prompt caching` 特性。借助 prompt caching，你不需要为每个 chunk 都重复传入整篇参考文档；你只需先把整篇文档缓存一次，之后在处理各个 chunk 时直接引用这份已缓存内容即可。

假设 chunk 大小为 800 tokens，文档长度为 8k tokens，context 生成指令占 50 tokens，每个 chunk 生成的上下文平均为 100 tokens，那么生成带上下文 chunk 的一次性成本大约是每一百万文档 tokens 1.02 美元。

## 方法论

我们在多个知识领域中进行了实验，包括代码库、小说、ArXiv 论文和科学论文，并比较了不同的 embedding 模型、检索策略与评估指标。附录 II 中给出了一些各领域问答样例。

下面的图展示了在所有知识领域上的平均表现，所采用的是效果最好的 embedding 配置（`Gemini Text 004`），并且检索 top-20 chunks。我们使用 `1 - recall@20` 作为评估指标，它衡量的是“相关文档没有出现在前 20 个结果中的比例”，也就是检索失败率。完整结果可以在附录中看到。无论在我们评估的哪一种 embedding 来源组合中，加入 contextualization 都会带来提升。

## 性能提升

我们的实验结果表明：

- `Contextual Embeddings` 将 top-20 chunk 的检索失败率降低了 35%，从 5.7% 降到 3.7%；
- 将 `Contextual Embeddings` 与 `Contextual BM25` 结合后，top-20 chunk 的检索失败率降低了 49%，从 5.7% 降到 2.9%。

也就是说，`Contextual Embedding + Contextual BM25` 的组合可以将 top-20 chunk 的检索失败率降低 49%。

## 实现时的注意事项

在实现 `Contextual Retrieval` 时，有几个问题需要重点考虑：

- `Chunk 边界`：文档如何切 chunk 会影响检索表现。chunk 大小、边界选取和 chunk 重叠方式都可能影响结果。<sup>1</sup>
- `Embedding 模型`：虽然 `Contextual Retrieval` 在我们测试过的所有 embedding 模型上都能提升表现，但有些模型收益更大。我们发现 Gemini 和 Voyage 的 embeddings 尤其有效。
- `自定义 contextualizer prompt`：我们提供的通用 prompt 已经很好用，但如果你根据特定领域或具体用例定制 prompt，例如加入知识库中其他文档才定义过的术语表，可能还会进一步提升效果。
- `Chunk 数量`：往上下文窗口里放入更多 chunk，的确更有可能覆盖到真正相关的信息；但信息过多也可能反过来干扰模型，所以这存在一个上限。我们测试了传 5、10 和 20 个 chunks，结果显示 20 个表现最好，但仍然建议你在自己的用例上做实验。

始终要运行 evals。响应生成阶段，也可能受益于直接传入 contextualized chunk，并显式区分“哪部分是上下文，哪部分是原始 chunk”。

## 通过 Reranking 进一步提升表现

最后一步，我们还可以把 `Contextual Retrieval` 与另一项技术结合起来，获得进一步提升。在传统 RAG 中，AI 系统首先会从知识库中检索出“可能相关”的信息块。对于大型知识库，这一步通常会返回很多 chunk，有时甚至上百个，而这些 chunk 的相关性和重要性并不相同。

`Reranking` 是一种常见的过滤技术，它能够确保真正最相关的 chunk 才会传给模型。Reranking 不仅会提升回答质量，也会降低成本和延迟，因为模型最终要处理的信息更少。其关键步骤如下：

- 先做一次初步检索，拿到最有可能相关的 chunks，我们实验中使用的是前 150 个；
- 将这些 top-N chunks 和用户查询一起传给 reranking 模型；
- 由 reranking 模型根据 chunk 与 prompt 的相关性和重要性打分，再选出 top-K，我们实验中使用的是前 20 个；
- 把 top-K chunks 作为上下文传给模型，生成最终结果。

将 `Contextual Retrieval` 与 reranking 结合，可以最大化检索准确率。

## 性能提升

市面上有多个 reranking 模型。我们的实验使用的是 Cohere 的 reranker。Voyage 也提供了 reranker，但我们还没有来得及测试。实验结果表明，在多个领域中，加入 reranking 步骤都能进一步优化检索。

具体来说，我们发现 `Reranked Contextual Embedding + Contextual BM25` 可以将 top-20 chunk 的检索失败率降低 67%，也就是从 5.7% 降到 1.9%。

## 成本与延迟上的考虑

在使用 reranking 时，一个重要问题是它对延迟和成本的影响，尤其是当你要对大量 chunk 做 rerank 时。因为 reranking 会在运行时额外增加一个步骤，所以即便 reranker 会并行给所有 chunk 打分，它仍然不可避免地会带来少量额外延迟。

因此，这里存在一个天然权衡：对更多 chunk 进行 rerank，通常能换来更好的表现；而对更少 chunk 进行 rerank，则能降低延迟与成本。我们建议你在自己的场景中尝试不同配置，找到适合的平衡点。

## 结论

我们运行了大量测试，比较了文中提到的所有技术组合：embedding 模型、是否使用 BM25、是否使用 contextual retrieval、是否使用 reranker、以及最终检索 top-K 的数量，并在多种不同数据集类型上做了验证。我们得到的结论如下：

- `Embeddings + BM25` 比单独使用 embeddings 更好；
- 在我们测试过的模型中，Voyage 和 Gemini 的 embeddings 表现最好；
- 将 top-20 chunks 传给模型，比只传 top-10 或 top-5 更有效；
- 为 chunk 添加上下文会显著提升检索准确率；
- 使用 reranking 比不使用更好；
- 这些收益是可以叠加的。要最大化表现，最佳组合是：使用 Voyage 或 Gemini 的 contextual embeddings，结合 contextual BM25，再加上 reranking，并将 20 个 chunks 一起放入 prompt。

我们鼓励所有构建知识库应用的开发者，使用我们的 cookbook 来尝试这些方法，从而解锁新的性能上限。

## 附录 I

下面给出了不同数据集、不同 embedding 提供方、是否叠加 BM25、是否使用 contextual retrieval，以及是否使用 reranking 时，在 `Retrievals @ 20` 条件下的结果拆解。

`Retrievals @ 10` 和 `@ 5` 的详细拆解，以及各数据集示例问答，可参考附录 II。

## 致谢

研究与写作由 Daniel Ford 完成。感谢 Orowa Sikder、Gautam Mittal 和 Kenneth Lien 提供关键反馈，Samuel Flamini 实现了 cookbooks，Lauren Polansky 负责项目协调，Alex Albert、Susan Payne、Stuart Ritchie 和 Brad Abrams 共同塑造了这篇博客文章。

<sup>1</sup> 文中此处为作者注记，强调 chunk 划分方式本身会影响检索效果。
