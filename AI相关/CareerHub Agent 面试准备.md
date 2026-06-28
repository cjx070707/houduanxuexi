# CareerHub Agent 面试准备

> 基于真实代码的架构拆解 + 面试拷打应答。所有描述均对应实际实现，不造概念。

---

## 一、一句话定位

**这不是一个 chatbot，是一个真正的 LLM function calling ReAct Agent。**

核心区别：
- 普通"Agent"项目：intent classifier → 预定工具链 → LLM 生成回答。LLM 走的是人预设好的路。
- 本项目：LLM 看到所有工具的 schema，**自主决定调哪些、什么顺序、传什么参数**。每一步都是真正的推理决策，不是 router。

---

## 二、整体架构

### 2.1 请求完整链路

```
用户消息
  │
  ▼
Fast Gate（纯 regex，<15ms）
  │ is_chitchat=True → simple_chat → 返回
  │ is_chitchat=False
  ▼
Task Classifier（6 种 TaskFamily）
  │
  ▼
上下文加载
  ├── 活跃目标 (GoalService)
  ├── 对话历史 (MemoryService, session-filtered)
  ├── 跨会话语义 (MemoryVectorService, Chroma)
  ├── 长期摘要 (SummaryService)
  ├── 用户偏好 (UserProfileService)
  └── AgentState 快照 (DB check + missing_info 推断)
  │
  ▼
System Prompt 构建
  (goals + profile + AgentState 全部注入)
  │
  ▼
ReAct 循环 (MAX_ITERATIONS = 6)
  ├── LLM chat_with_tools / stream_chat_with_tools
  ├── tool_calls 存在 → 执行工具 → 结果追加 messages → 继续
  ├── tool_calls 不存在 → Action Claim 验证
  │     ├── 声称了行动但没调工具 → 注入修正 → continue
  │     └── 无问题 → break，输出 final answer
  └── 超出 MAX_ITERATIONS → 返回兜底回答
  │
  ▼
持久化 & 后台任务（异步）
  ├── save_turn (SQLite)
  ├── memory_vector.index_turn (Chroma)
  ├── update_profile (偏好提取)
  ├── career_events.sync_from_message
  └── maybe_compress (超 24 条触发摘要压缩)
  │
  ▼
AutonomousAgentResult → SSE 流式推送前端
```

### 2.2 Fast Gate

- 判断逻辑：**纯硬规则，不走 LLM**
- 条件：消息 `len() > 15` → 直接 `is_chitchat=False`
- `≤15` 字 + 匹配寒暄 regex（你好/谢谢/再见等）→ `is_chitchat=True`
- 意义：节省 3-8s LLM 延迟，不为"你好"走完整 ReAct

### 2.3 TaskFamily 分类

| Family | 触发关键词方向 | 作用 |
|--------|--------------|------|
| JOB_SEARCH | 找岗位、实习、职位 | Phase 1 历史过滤权重 |
| RESUME | 简历、gap、技能差距 | 推断缺简历时的 missing_info |
| APPLICATION | 投递、申请、跟进 | 历史过滤 |
| INTERVIEW | 面试、笔试、反馈 | 历史过滤 |
| GOAL | 目标、计划、进展 | 历史过滤 |
| GENERAL | 其他 | 默认 |

影响：`score_turn()` 对历史消息打分，同族历史优先进入上下文 budget。

---

## 三、记忆系统（4 层）

这是面试最容易被追问的部分，要能说清楚每层做什么、为什么这样设计。

### 3.1 短期/Session 记忆

- **存储**：SQLite `conversation_turns` 表
- **加载逻辑（两阶段预算，总上限 8 条）**：
  - **Phase 1**：当前 session，`score_turn()` 按 task_family 关键词过滤，锚点（最近 4 条）必选，旧 turn 按评分填充剩余 budget
  - **Phase 2**：Chroma 跨 session 语义召回，排除当前 session，最多 2 条

```python
_ANCHOR_TURNS = 4    # 最近 2 轮对话，无论相关性
_HISTORY_BUDGET = 8  # 总上限
_SEMANTIC_BUDGET = 2 # 跨 session 语义
```

- **设计意图**：不是简单取最近 N 条，而是"任务相关性"优先，防止无关历史污染上下文

### 3.2 长期摘要记忆

- **触发条件**：`count_turns > ARCHIVE_THRESHOLD (24)`
- **压缩流程**：
  1. 找出候选 turn（keep_ids 外的历史）
  2. 拼接候选文本 + 已有摘要
  3. LLM 压缩（system prompt 要求：保留求职目标/工具结果/关键决策/进展，≤400 字）
  4. 更新 `conversation_summaries` 表，**删除**原始候选 turn
- **消费**：注入 system prompt 的 `【历史对话摘要】` 段落
- **关键设计**：删除原始 turn，节省 token；LLM 选择性保留"值得记住的事实"

### 3.3 跨会话向量记忆

- **向量库**：ChromaDB PersistentClient
- **Embedding**：DashScope text-embedding-v3（1024 维），无 API 时 fallback 到 token hash
- **Index**：每轮结束后调 `index_turn(turn_id, user_id, session_id, role, content)`
- **Search**：
  ```python
  where = {
    "$and": [
      {"user_id": {"$eq": user_id}},    # 用户隔离
      {"session_id": {"$ne": exclude_session}}  # 排除当前 session（Phase 1 已覆盖）
    ]
  }
  ```
- **作用**：跨会话的"你上次说你在找 Sydney 后端岗"这类语义记忆

### 3.4 用户 Profile（偏好记忆）

- **存储**：SQLite `user_profiles` 表，JSON 格式
- **字段**：location / industry / job_type / salary / timeline / avoid / other
- **更新机制**：每轮 after-turn 后台调 `update_profile()`，LLM 从对话中提取偏好 JSON，**新值覆盖旧值，null 不覆盖**
- **消费**：注入 system prompt 的 `【用户已知偏好】` 段落
- **作用**：用户只需说一次"我在 Sydney"，后续对话不需要重复

### 3.5 AgentState 快照（per-turn）

每轮请求时 `build()` 一次，查 DB 生成状态：

```
【Agent 状态快照】
• 任务类型：job_search
• 数据：简历=已上传 ｜ 投递=3条 ｜ 面试=0条 ｜ 目标=1个
• 用户已知：Python、Sydney、junior
⚠️ 缺失：暂无面试记录，无法提供面试反馈分析
```

- `missing_info`：规则推断，如 JOB_SEARCH + 无简历 → "简历尚未上传"
- 注入 system prompt，让 LLM 在回答前就知道用户的完整状态

---

## 四、工具体系

### 4.1 工具注册机制

所有工具通过 `ToolRegistry` 统一注册，每个工具是一个 `ToolDefinition`：

```python
ToolDefinition(
    name="search_jobs",
    description="...",          # LLM 看到的触发描述
    category="job_search",
    input_model=SearchJobsInput, # Pydantic 校验
    handler=fn,                  # 实际执行函数
)
```

LLM 通过 function calling 选择工具，`ToolRegistry.run(name, args)` 执行。

### 4.2 工具清单

| 工具 | 功能 | 关键技术 |
|------|------|---------|
| `search_jobs` | 岗位搜索 | Hybrid RAG（ChromaDB + BM25 + RRF） |
| `get_resume` | 读取用户简历 | SQLite JOIN candidates |
| `analyze_gap` | 简历 vs JD 差距分析 | LLM 结构化输出 |
| `match_resume_to_jobs` | 简历与岗位匹配打分 | LLM 评分 |
| `get_goals` | 查询求职目标 | SQLite goals 表 |
| `set_goal` | 设定新目标 | 含截止时间，持久化 |
| `log_progress` | 记录目标进展 | goal_progress 表 |
| `update_goal_status` | 标记目标完成/放弃 | status 字段更新 |
| `get_candidate_profile` | 读取候选人画像 | candidates 表 |
| `get_applications` | 查询投递记录 | applications 表 |
| `log_application` | 记录投递/跟进 | 写入 applications |
| `get_interview_feedback` | 查询面试反馈 | interviews 表 |
| `get_career_insights` | 职业诊断 | CareerDiagnosisEngine |

### 4.3 Hybrid RAG 细节（search_jobs）

```
用户查询
  │
  ├── 向量召回：ChromaDB cosine similarity（语义）
  │
  └── 词法召回：rank_bm25（精确关键词，如 FastAPI、vLLM）
  │
  ▼
RRF 融合排序（Reciprocal Rank Fusion）
  score = Σ 1/(k + rank_i)，k=60
  │
  ▼
Top-K 岗位返回
```

**为什么需要 BM25**：纯向量对精确技术词（FastAPI、vLLM、K8s）召回不稳定；BM25 在词法匹配上是互补。

### 4.4 Action Claim 验证（防幻觉机制）

LLM 有时会在 final answer 里声称"已记录"但不调工具。解决方案：

```python
def _check_action_claims(answer, tool_trace):
    # 扫描 final answer 的行动声明 regex
    # 与 tool_trace 对比
    # 返回"声称了但没调"的工具列表

if uncalled and iteration < MAX_ITERATIONS - 1:
    # 注入修正消息，continue 而非 break
    messages.append({"role": "user", "content": "你声称已完成X但没调工具，请立即执行"})
    continue
```

**关键**：这是**结构性保障**，不是触发词枚举——任何表达方式都能被正则捕获，复合意图不受影响。

---

## 五、演示场景

### 场景 1：搜岗位（单工具）

```
用户：帮我找几个 Sydney 的 Python 后端实习
Agent 链路：search_jobs(query="Python backend intern Sydney")
```

展示：Hybrid RAG 召回，SSE 实时状态推送（"🔧 调用工具：search_jobs"）

### 场景 2：Gap 分析（多工具链）

```
用户：[粘贴 JD] 帮我看看我和这个岗位的差距
Agent 链路：get_resume → analyze_gap(resume, jd)
```

展示：自主多步推理，不是预定工具链

### 场景 3：复合意图（验证记忆 + 记录）

```
用户：分析完之后，帮我记一下我投了这个岗位
Agent 链路：analyze_gap → log_application
```

展示：同一轮对话完成两件事；如果 LLM 声称记录但没调工具 → 声明验证机制自动纠错

### 场景 4：跨会话记忆

```
第一次对话：我在找 Sydney 的 AI 岗
（session 结束）
第二次对话：帮我推荐适合我的职位
Agent：无需重新问地点，从 user_profile + 向量记忆中恢复偏好
```

展示：4 层记忆体系的实际价值

### 场景 5：职业诊断

```
用户：我投了很多岗位但没有进展，怎么办？
Agent 链路：get_career_insights → CareerDiagnosisEngine
输出：结构化诊断（瓶颈点 + 置信度 + 行动建议）
```

---

## 六、面试拷打应答

### Q1：你说这是真 ReAct，怎么证明？

**答**：证明点在 `autonomous_agent_service.py` 的循环逻辑：LLM 每次看到的是全量工具 schema（`tool_schemas = self._build_tool_schemas()`），response 里有 `tool_calls` 就执行，没有就退出。没有任何预定的工具顺序或 if-else 分支决定调哪个工具。用户说"分析差距顺便记录一下投递"，agent 会自主决定先调 `analyze_gap` 再调 `log_application`，这个顺序是 LLM 在推理，不是代码写死的。

### Q2：记忆系统用了 RAG，和普通 RAG 有什么区别？

**答**：有三个差异：
1. **两阶段 budget 管理**：不是简单取最近 N 条，Phase 1 按 task_family 关键词评分过滤，Phase 2 跨 session 语义补充。避免无关历史污染上下文。
2. **长期压缩**：超过 24 条 turn 触发 LLM 摘要，压缩后删原始数据。普通 RAG 只做检索，不做生命周期管理。
3. **用户 profile 提取**：RAG 是被动检索，profile 是主动提取——每轮对话后 LLM 自动提取用户偏好 JSON，后续注入 system prompt，不再需要检索就能"记住"用户是谁。

### Q3：Hybrid RAG 的 RRF 是怎么工作的？k=60 是什么意思？

**答**：RRF（Reciprocal Rank Fusion）公式是 `score = Σ 1/(k + rank_i)`。每个候选文档在各个召回器里都有一个排名，RRF 把多个排名融合为一个分数。k=60 是平滑参数，防止 rank=1 的文档分数过于主导——当 k 越大，头部和尾部文档的分数差异越小，融合结果越平滑。为什么选 60？这是 Cormack et al. 2009 原论文的经验值，在大多数场景下表现最优。

### Q4：SSE 流式输出怎么实现的？为什么不用 WebSocket？

**答**：SSE 够用且更简单。实现上的关键问题是：LLM 调用是同步阻塞的，但 FastAPI 是 async。解决方案：`asyncio.to_thread()` 把阻塞调用放线程池，用 `asyncio.Queue` + `loop.call_soon_threadsafe()` 做线程到协程的安全桥接。每个 token 通过 `on_token` callback 推进 queue，event_stream 协程从 queue 取数据发 SSE 事件。选 SSE 而不是 WebSocket：SSE 是单向推送，HTTP 长连接，不需要协议升级，反向代理（nginx）开箱即用，复杂度更低。

### Q5：你这个 MAX_ITERATIONS=6，会不会太少？

**答**：对当前工具集来说够用。实际上复杂查询最多 3-4 步（get_resume → analyze_gap → search_jobs → log_application）。6 是安全边际。更重要的是：无限循环的危害远大于截断——模型如果陷入循环会消耗大量 token 且用户体验极差。当前场景没有需要超过 6 步的合理工作流。如果业务扩展到需要更多步，可以动态调整或按 task_family 设置不同上限。

### Q6：工具调用失败了怎么办？

**答**：`ToolRegistry.run()` 外面有 try-except，失败返回 `{"error": str(exc)}`。这个 error 结果会作为 tool message 追加回 messages，LLM 在下一个 iteration 看到错误信息后可以选择：重试、换工具、或在 final answer 里告知用户。不会因为单个工具失败而崩掉整个 agent。同时 `tracer.log_tool_call()` 记录失败信息用于监控。

### Q7：Action Claim 验证的 regex 会不会误判？

**答**：两个设计防误判：
1. 模式足够具体——不是匹配"已记录"（太泛），而是"已记录.*投递"/"已记录你跟进"等带上下文的组合，普通陈述句不会触发。
2. 只在 `tool_trace` 里没有对应工具时才报警——如果工具确实被调用了，声明是合法的，不会干预。
误判的最坏结果是多走一次 loop（浪费一次 LLM 调用），不会导致错误数据写入或错误回复。

### Q8：用户数据隔离怎么做的？

**答**：三处隔离：
1. **SQL 层**：所有查询都带 `WHERE candidates.user_id = ?` / `WHERE user_id = ?`
2. **向量层**：Chroma search 的 `where` 条件包含 `{"user_id": {"$eq": user_id}}`，不会查到其他用户的向量
3. **Session 层**：历史加载按 `session_id` 过滤，跨 session 语义召回也排除其他用户

### Q9：简历视觉解析是怎么做的？

**答**：用户上传图片/PDF → 后端用 PyMuPDF 把 PDF 转成 JPEG（150 DPI）→ 调用 Qwen-VL（多模态）解析图片 → 输出结构化 JSON（ParsedResumeImage schema：name/email/phone/education/skills/experience/projects）→ 存 `resumes.parsed_json`。关键工程点：视觉 API 是同步阻塞调用，用 `asyncio.to_thread()` 包装，防止阻塞 FastAPI 事件循环。

### Q10：这个项目最大的局限是什么？

**答**：三个真实局限：
1. **记忆窗口浅**：6 轮滑动 + 摘要，无法跟踪几周前的细节对话；摘要会丢失细节
2. **分析深度依赖 LLM**：`analyze_gap` 没有独立评分模型，结果质量完全依赖 Qwen 的理解能力，无法做 A/B 对比验证
3. **岗位数据是本地样本**：`search_jobs` 基于手工 seed 的 55 条数据，没有接真实招聘源（Adzuna API 已集成但数据量有限）

---

## 七、技术栈速查

| 组件 | 技术 |
|------|------|
| LLM | DashScope Qwen（qwen3.5-plus，OpenAI-compatible API） |
| Embedding | DashScope text-embedding-v3（1024 维） |
| 向量检索 | ChromaDB PersistentClient |
| 词法检索 | rank_bm25 |
| 融合排序 | RRF（k=60） |
| 多模态 | Qwen-VL（简历图片解析） |
| 后端 | FastAPI + SQLite |
| 流式输出 | SSE + asyncio.Queue + call_soon_threadsafe |
| 前端 | React 18 + TypeScript + Vite |
| PDF 处理 | PyMuPDF（fitz） |
| 测试 | pytest（481 个 case） |

---

*最后更新：基于实际代码，非设计稿。*
