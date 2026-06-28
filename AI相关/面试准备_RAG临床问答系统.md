# 面试准备：基于 RAG 的临床指南智能问答系统

---

## 一、项目介绍

### 一句话版本

基于 NICE NG198 痤疮临床指南构建的 Agentic RAG 问答系统，核心是 guideline-first 分层检索 + LangGraph Agent 决策，并建立了 retrieval / grounded QA / support governance / refusal 四层评测体系。

---

### 30 秒版（开场白）

这是一个面向临床知识检索的 Agentic RAG 项目，不是普通的知识库问答。核心设计是 guideline-first：优先从 NICE 官方 recommendation 检索证据，必要时才引入 supporting evidence 文档补充，证据仍然不足则拒答。

整体是一个两阶段 Agent 流程：第一轮只查主 guideline，证据不足就 query rewrite 再做第二轮 main + support 检索，由 LangGraph 控制 answer / rewrite / refuse 三个动作。

检索层做了 reranker、metadata filtering 和 query routing 三轮优化，并建立了四层评测体系去验证每一轮优化的真实效果。

---

### 2 分钟版（主动讲）

**背景**：临床场景里，guideline recommendation 和 support 文档的证据权威性不同。如果混检，support 里 lexical overlap 高但 authority 低的内容很容易污染 top-k，导致生成答案用的不是正式 recommendation。

**系统设计**：

离线阶段，我对 NICE NG198 guideline 和 supporting evidence review 分别做清洗、按 recommendation block 切块、embedding，构建 main 和 support 两套 FAISS 索引。

在线阶段是 guideline-first 的两阶段检索 + LangGraph Agent：
- 第一轮只查 main guideline，候选经 Cross-Encoder Rerank 后送 evidence judge
- 证据充分 → 直接生成带 citation 的回答
- 证据不足 → query rewrite → 第二轮 main + support 检索
- 第二轮仍不足 → 拒答

**检索优化**：做了三轮实验：
1. Hybrid retrieval（BM25 + dense + RRF）：没有超过 dense baseline，说明当前语料已经很结构化，lexical 信号更多是扰动
2. Metadata filtering：对 recommendation-level 排序有正收益，减少错误召回比增加更多信号更有效
3. Query routing：retrieval 指标与 metadata filtering 持平，但 grounded QA 提升明显（79% → 96%），说明收益在系统决策路径层，而不只是召回层

**评测设计**：不只看 Hit@k，分四层：retrieval / grounded QA / support governance / refusal boundary，分层定位问题出在召回、排序、证据治理还是生成。

---

## 二、项目核心技术点

### 2.1 为什么要 guideline-first，不直接混检

guideline 是正式 recommendation，support 更多是证据讨论和 rationale。混检时，support 文档因为 lexical overlap 高会抢占 top-k，但它的 authority 低。

所以我做了双索引 + 分层检索路径：第一轮只查 main，引入 support 是有条件的，且 support governance 是单独的评测维度。

### 2.2 为什么要 Agent，不用单轮 RAG

单轮 RAG 默认"检到了就答"，在医疗 guideline 场景里不够。更关键的问题是"当前证据够不够"。

所以引入 LangGraph Agent：evidence judge → answer / rewrite / refuse 三个分支。这让系统能拒答，而不是在证据不足时乱答。

### 2.3 Query Rewrite vs Query Routing 的区别

- **Query Rewrite**：第一轮证据不足后，把用户原始问题改写成更 retrieval-friendly 的形式，再检一次
- **Query Routing**：在检索前，根据 query 类型决定走哪条检索路径（是否开 metadata filtering、第二轮是否引入 support）

一个是改 query，一个是分 path，作用阶段不同。

### 2.4 Metadata Filtering 怎么做的

基于 question type（推断出的）和 chunk 的 section metadata 做 soft filtering。不是 hard filter 过滤掉，而是作为 rerank 的加权信号，降低明显不相关 section 的候选排名。

效果：Page Hit@3 = 0.964，Rec MRR@3 从 0.845 提升到 0.872。

### 2.5 最终指标

**Retrieval（query routing + metadata filtering）**：
- Page Hit@3 = 0.964，Page MRR@3 = 0.943
- Rec Hit@3 = 0.911，Rec MRR@3 = 0.872

**Grounded QA**：
- Action accuracy = 23/24 = **0.958**（baseline 0.792）
- No forbidden claims = 24/24 = 1.000
- Primary-source pass = 23/24 = 0.958

**Refusal Boundary**：
- Refusal accuracy = 16/20 = 0.800

---

## 三、面对拷打

### Q1. 为什么 hybrid retrieval 没有提升

这个语料是结构化的 NICE guideline recommendation block，dense baseline 本来就很强（Hit@3 = 0.964）。BM25 的 lexical 信号在这里没有提供有效的新召回，反而在排序上引入了扰动。

所以 hybrid 在这个场景下边际收益很低。**我没有为了技术栈好看而保留它。**

---

### Q2. 你的 evidence judge 靠 LLM，不怕不稳定吗

怕，所以我不把它包装成完全可靠的组件。当前它是一个启发式决策节点，不是形式化 verifier。

我通过两个方式降低风险：一是限制决策空间，只有 answer / rewrite / refuse 三个动作；二是用 refusal boundary 和 support governance 两个评测维度去测它的系统级行为是否符合预期。

后续改进方向：规则 + 模型结合，减少对单次 LLM 判断的依赖。

---

### Q3. 怎么证明 metadata filtering 和 query routing 真的有效

我没有只看单一指标。

- Metadata filtering：在 retrieval ranking 上带来了 Rec MRR@3 的提升，说明 recommendation-level 排序更准了
- Query routing：retrieval 指标和 metadata filtering 接近，但 grounded QA Action accuracy 从 0.792 提升到 0.958，说明它的收益在系统决策路径层，而不是召回层

两个优化解决的是不同层的问题，所以要分层评测才能看清楚。

---

### Q4. 为什么不直接用更强的大模型 end-to-end 回答

这个场景的核心问题不是生成能力，而是证据边界和可追溯性。

如果没有 retrieval policy、support governance 和 refusal control，只换更强的模型，只会让"答错时更像对的"。guideline-first 的设计约束更重要。

---

### Q5. 你的 benchmark 数据量不大，结果可信吗

这是事实，current benchmark 规模不大（retrieval 56 题，grounded QA 24 题等）。

它更像一个结构完整的实验框架，而不是大规模生产 benchmark。结果能支持方向性判断（比如 hybrid 无效、routing 有效），但不能声称已经充分覆盖所有场景。

---

### Q6. 你的 query routing 是不是太规则化了，很弱

是，v1 是规则版，不是学习式分类器。这是刻意的。

我先想验证"按问题类型分流"这个策略本身有没有价值，再决定要不要升级成学习式 router。实验结果证明策略有效，所以后续有理由继续投入。先验证方向，再升级实现。

---

### Q7. 这个项目最弱的地方是什么

两个地方：

1. **Evidence judge 形式化程度不够**：当前完全依赖 LLM 判断，没有规则兜底，稳定性有上限
2. **Must-include pass 偏低**：grounded QA 里 must-include pass = 10/24，说明回答完整性还不稳定，关键 recommendation 有时没有被覆盖到

这两个问题是系统目前最值得继续优化的方向。

---

### Q8. 简历里写的"multi-stage retrieval"具体是什么

就是两阶段检索：
- 第一阶段：只查 main guideline，用 rerank + evidence judge 判断是否足够
- 第二阶段：query rewrite 后查 main + support，再 judge 一次

两阶段的分隔点是 evidence judge 的判断结果，不是固定轮次。

---

### Q9. Citation 怎么做的，怎么保证生成内容被 citation 支持

Answer 生成时，context 里带了 chunk 来源（page / recommendation ID）。Prompt 里要求模型只引用传入 context 里的内容，并在回答里标注来源。

当前是 prompt-level 约束，不是形式化 grounding verifier。`No forbidden claims = 1.000` 说明在当前 benchmark 上没有出现越出证据的情况，但这不等于完全杜绝幻觉，只是说明当前 prompt 设计有效。

---

### Q10. 这个项目怎么收尾（被问"你这个项目最大的收获是什么"）

这个项目让我比较系统地把 RAG 从"能跑"推进到"能实验、能评测、能解释"。

最重要的收获不是加了几个技术，而是建立了一个完整的实验闭环：有 baseline、有对照实验、有分层评测、有可解释的结论。这让我能说清楚"哪个优化真的有用、为什么有用、哪里还不够"，而不只是堆技术栈。

---

## 四、你最该练熟的 6 个问题

1. 你这个项目解决什么问题？为什么要 guideline-first？
2. Agent 流程是怎么设计的？为什么不用单轮 RAG？
3. 为什么 hybrid retrieval 没有提升？
4. Metadata filtering 和 query routing 分别解决什么问题？
5. 你怎么评测？为什么要分四层？
6. 这个系统现在最弱的地方是什么？

---

## 五、关键数字备查

| 指标 | Baseline | 最终（query routing v1） |
|------|----------|--------------------------|
| Page Hit@3 | 0.964 | 0.964 |
| Page MRR@3 | 0.923 | 0.943 |
| Rec Hit@3 | 0.893 | 0.911 |
| Rec MRR@3 | 0.845 | 0.872 |
| Grounded QA Action Acc | 0.792 | **0.958** |
| No Forbidden Claims | 1.000 | 1.000 |
| Refusal Accuracy | 0.800 | 0.800 |
