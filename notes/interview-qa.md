# Agent 开发面试题 × Codex 源码

把小红书 / 牛客 / CSDN 上流传的 agent 开发面经题目，逐条对照本项目源码作答。

**前提说明**：Codex 是 **harness（智能体执行框架）**，不是业务 agent（内容审核、智能客服）。所以像「意图分流」「query 模糊还是清晰」这类题，它的答案会是「我们不做」——而这个「不做」本身是有论证的，往往比硬答更有说服力。

## 目录

- [Q1. Harness 层是什么？包括哪些内容？](#q1)
- [Q2. Skill 和 Tool 的区别？Skill 什么时候 fetch？](#q2)
- [Q3. 记忆模块怎么设计？](#q3)
- [Q4. 工具调用超时 / 异常的容错设计](#q4)
- [Q5. 大模型返回 JSON 格式异常，链路报错怎么处理？](#q5)
- [Q6. 多 Agent 并发控制 / 资源竞争](#q6)
- [Q7. 上下文压缩 / 长对话](#q7)
- [Q8. 为什么自研而不用 LangChain / LangGraph？](#q8)
- [Q9. 如何判断 query 是模糊还是清晰？](#q9)
- [Q10. 模型一致性 & Token 成本](#q10)
- [Q11. 其他题目速查表](#q11)
- [附：这份面经本身的可信度](#appendix)

---

<a id="q1"></a>
## Q1. Harness 层是什么？包括哪些内容？

面经的标准框架是「约束执行空间 + 可观测性 + 反馈闭环」。Codex 的 harness 可以切出六块：

| 层 | Codex 实现 |
|---|---|
| 输入组装 | `core/src/context/`（AGENTS.md、环境、skills catalog）、`prompt` 模块 |
| 决策 | `tools/router.rs:244` `build_tool_call`（唯一分类器）、`session/turn.rs` 的 `needs_follow_up`（唯一循环闸门） |
| 执行 | tool runtime + `execpolicy` + `tools/approvals.rs` + `sandboxing` |
| 记忆 | history + rollout + `compact` + `memories` |
| 观测 | `otel`、`call_trace`、`tool_dispatch_trace`、`rollout-trace`（`replay_bundle` 可回放） |
| 配额 | `TokenBudget`、`RolloutBudget`、`AgentRegistry` 槽位 |

**可直接用的答法**：harness 的职责不是让模型更聪明，而是**把模型的不确定性收敛成确定的结果**。判断 harness 好不好看两件事——模型胡来时你能不能兜住（错误回灌），模型要动手时你能不能拦住（审批 + 沙箱）。

---

<a id="q2"></a>
## Q2. Skill 和 Tool 的区别？Skill 什么时候 fetch？

详细展开见 [skills-vs-tools.md](skills-vs-tools.md)。核心答案：

**① Discovery 阶段一样，Execution 阶段完全不同。**

- **Tool** = 可执行能力。`ToolSpec`（给模型看的 schema）+ `ToolExecutor`（真正跑的实现），调用后返回值进对话历史。
- **Skill** = 知识/流程。`SKILL.md` + frontmatter（`codex-rs/skills/src/model.rs:8` 的 `SkillMetadata`），**自己不执行任何东西**，靠 `dependencies` 里声明的工具落地。

**② 什么时候 fetch —— 不是「下一轮」，是三条路径：**

| | 触发源 | 谁去读正文 | 生效时机 |
|---|---|---|---|
| 目录（name+desc+path） | 会话开始 | — | **每一轮**都在 |
| 路径 ① `$skill-name` | 用户消息 | **harness** | **当轮**请求前 |
| 路径 ② 语义自匹配 | 模型判断 | **模型（工具调用）** | 工具返回后的**下一轮** |

**③ 为什么省 token —— 机制层答案：**

目录常驻（`role: developer`，`content_kind: skills.catalog`，`fragments.rs:41`），正文按需（`role: user`，`<skill>` 标记，`fragments.rs:78`）。

> 省 token 靠的不是压缩，是**把「有没有」和「是什么」拆成两次不同成本的访问**。推的是索引，拉的是内容。

补充加分：**目录本身也有预算**（context window 的 2%，上限 10k token，`render.rs:127-152`），超预算会逐字符削描述，再超就整个技能条目丢弃。

---

<a id="q3"></a>
## Q3. 记忆模块怎么设计？

面经的分类框架是**工作记忆 / 情景记忆 / 语义记忆 / 程序记忆**，写入走「感知→判断→提炼→存储」，检索用 Recency + Relevance + Importance 加权。

Codex 的映射：

| 面经分类 | Codex 实现 |
|---|---|
| 工作记忆 | context window / `Session` 的对话历史 |
| 情景记忆 | `rollout`（append-only 事件日志，带 `turn_id`、时间戳） |
| 语义记忆 | `memories`（两阶段提炼产物） |
| 程序记忆 | `AGENTS.md` + `SKILL.md` + `execpolicy` 规则 |

**写入管线恰好是两阶段**，能逐字对上「提炼→整合」：

- **Phase 1**（`codex-rs/memories/write/src/phase1.rs`）：从 rollout 抽取候选记忆，产出 `StageOneOutput`，带结构化 `output_schema()`。写前脱敏：`sanitize_response_item_for_memories` + `codex_secrets::redact_secrets`。
- **Phase 2**（`phase2.rs`）：consolidation（整合/反思）。`build_consolidation_prompt_for_version` 合并去重、`validate_consolidation_artifacts_for_version` 校验产物、**`prune_old_extension_resources` 主动遗忘**。整个过程在独立 workspace 里做（`memory_workspace_diff`）。

**读写故意解耦**：`codex-rs/memories/read/src/lib.rs` 的 crate 注释写明「intentionally does not depend on the memory write pipeline」。

> 这点值得展开：读写分离不只是代码整洁，是因为两者的**失败模式完全不同**——写入是低频批处理、可重试；读取在关键路径上、必须快且绝不能阻塞。

---

<a id="q4"></a>
## Q4. 工具调用超时 / 异常的容错设计

详细展开见 [fault-tolerance.md](fault-tolerance.md)。逐条对照面经标准答案：

**① 指数退避 + jitter** —— `core/src/util.rs:86`，`200ms × 2^(n-1) × jitter(0.9..1.1)`。正好是面经说的 `2^i`，**但多了 jitter**，这是加分点：

> 裸指数退避会造成重试风暴同步（thundering herd），所有客户端在同一时刻撞上服务端。±10% 抖动把尖峰打散。

**② 优先尊重服务端信号** —— `responses_retry.rs:112`：`err.retry_delay().unwrap_or_else(|| backoff(retry_count))`。服务端给了 `retry-after` 就听服务端的。

**③ 降级** —— WebSocket→HTTPS，且**降级后重置重试计数**（`retries = 0`），因为换传输等于换了失败面。

**④ 异常分类** —— `protocol/src/error.rs:372` `is_retryable()` 穷尽匹配。原则：**网络与容量问题重试，语义与策略问题不重试**。

**⑤ 熔断器？—— 没有。** 诚实的答法：本地 CLI 场景单用户单进程，不存在下游雪崩，用 `max_retries` + 用户可见提示代替；做成多租户服务必须补。

**⑥ 让不让模型参与错误恢复？** 分类处理：

| 情况 | 处理 | 效果 |
|---|---|---|
| 参数解析失败 | `RespondToModel` | 错误进历史，**模型自己改** |
| 工具执行失败 | `FunctionCallOutput { success: false }` | 同上 |
| 工具 task panic | `Fatal` | **终止整个 turn** |

**主动指出设计问题**（面试加分）：`parallel.rs:242` 把 `JoinError` 一律提升成 `Fatal`，等于放弃了这条路最该发挥作用的场景——工具崩了本可以让模型换个打法重试。`err.is_panic()` 本可以走 `RespondToModel`。

---

<a id="q5"></a>
## Q5. 大模型返回 JSON 格式异常，链路报错怎么处理？

分四层：

**① 不预解析** —— `protocol/src/models.rs:1069-1071`，`FunctionCall.arguments` 刻意保持 `String`：

```rust
// The Responses API returns the function call arguments as a *string* that contains
// JSON, not as an already‑parsed object. We keep it as a raw string here and let
// Session::handle_function_call parse it into a Value.
arguments: String,
```

**② 解析失败是可回报的错误，不是崩溃** —— `build_tool_call` 返回 `Result<Option<ToolCall>, FunctionCallError>`，失败走 `RespondToModel`。

**③ 结构化约束优先** —— 工具 schema 走 `ToolSpec`（JSON Schema），不靠 prompt 要求模型「输出 JSON」。面经标准答案的四个手段（Prompt + few-shot / JSON Schema / 容错解析 / 降级兜底），Codex 用了后两个，因为前两个在平台层保证了。

**④ 兜底** —— 不认识的 item 落到 `_ => Ok(None)`，当作普通输出收尾。**永远不会因为模型返回意外类型而崩。**

---

<a id="q6"></a>
## Q6. 多 Agent 并发控制 / 资源竞争

详细展开见 [multi-agent.md](multi-agent.md)。三层闸门：

**① 槽位 + 深度**

`agent/registry.rs`：`reserve_spawn_slot(agent_max_threads)` / `try_increment_spawned` / `release_spawned_thread`；深度用 `exceeds_thread_spawn_depth_limit(child_depth, max_depth)`（`registry.rs:91`）。

超深度直接把错误回给模型（`multi_agents/spawn.rs:70-74`）：

```rust
return Err(FunctionCallError::RespondToModel(
    "Agent depth limit reached. Solve the task yourself.".to_string(),
));
```

**② 并行 / 串行的读写锁分层** —— 最值得讲的一层（`tools/parallel.rs:109`、`158-161`）：

```rust
let supports_parallel = router.tool_supports_parallel(&call);
// ...
let guard = if supports_parallel {
    Either::Left(lock.read().await)      // 只读类工具：共享锁，真并行
} else {
    Either::Right(lock.write().await)    // 副作用类工具：独占锁，串行
};
```

> **不是全局串行，也不是无脑并行**——用 `RwLock` 按工具是否只读分层。

**③ 等待超时** —— `DEFAULT_WAIT_TIMEOUT_MS = 30_000`，`timeout_ms` 被 clamp 到 `[MIN, MAX]`。

**④ 两道闸别搞混**：

| | 限什么 | 检查时机 |
|---|---|---|
| `AgentRegistry.total_count` | 活跃 agent **总数** | spawn 时 |
| `AgentExecutionLimiter` | **同时在跑 turn** 的 agent 数 | turn 开始时 |

前者是「你有几个 agent」，后者是「同时有几个在烧 token」。

---

<a id="q7"></a>
## Q7. 上下文压缩 / 长对话

**① 双阈值，不是单阈值** —— `ContextWindowTokenStatus` 同时维护 `auto_compact_scope_limit` 和 `full_context_window_limit`，区分「该开始整理了」和「要撑爆了」。

**② 预判式压缩** —— `session/turn.rs:179`：

```rust
// TODO(ccunningham): Pre-turn compaction runs before context updates and the
// new user message are recorded. Estimate pending incoming items (context
// diffs/full reinjection + user input) and trigger compaction preemptively
// when they would push the thread over the compaction threshold.
```

**在拼装 prompt 之前就估算本轮要新增的 token**，提前压——而不是等请求失败再补救。

**③ 不是原地压缩，是「开新窗口」** —— `should_roll_over` + `take_new_context_window_request()`，压缩产物带 `CompactedHistoryMetadata { window_number, window_ids, compaction_response_id, compaction_model_hash }`。历史是**多窗口可回溯**的结构，旧窗口还在 rollout 里可以回放。

**④ 压缩本身也会失败** —— 独立的 `compact_model_fallback.rs`，压缩用哪个模型、降级到哪个都有 fallback。

**⑤ 一个诚实的取舍** —— `session/turn.rs:611`：

```rust
// as long as compaction works well in getting us way below the token limit, we shouldn't worry about being in an infinite loop.
```

没有强制终止循环的保护，赌的是压缩能把 token 降得足够低。聊到「死循环 / 成本失控」时这是可以被追问的点。

---

<a id="q8"></a>
## Q8. 为什么自研而不用 LangChain / LangGraph？

用这个代码库回答最有说服力——**因为 Codex 的编排层薄得惊人**。

整个 agent loop 就是 `session/turn.rs` 里 `run_turn` 里的一个 `loop`，加一个 `needs_follow_up` 布尔量。**没有 DAG 引擎、没有节点/边、没有状态机框架。** LangGraph 的「点」和「边」在这里退化成了一次 `loop` 迭代。

而真正复杂的部分——审批策略、execpolicy 规则语言、三平台沙箱（Landlock / Seatbelt / Windows）、多 Agent 配额、上下文窗口翻滚、rollout 回放、MCP——**没有任何框架提供**。

**可直接用的答法**：

> 框架解决的是「怎么把节点连起来」，而 agent 的难点从来不在连线，在于**怎么把模型的不确定性关进笼子**。Codex 选择自研，是因为它的复杂度 90% 在 harness 外围（沙箱、审批、平台差异），只有 10% 在编排——用框架会让那 10% 变简单，但那 10% 本来就不是瓶颈。

**再加两点 Rust 特有的论据**：

- `ResponseItem` 是强类型枚举，`build_tool_call` 是穷尽 `match`——**新增 item 类型时编译器会强制你处理**。
- `JoinError` / `Cancellation` / `AbortOnDropHandle` 这套生命周期语义，Python 框架给不了。

---

<a id="q9"></a>
## Q9. 如何判断 query 是模糊还是清晰？

这题面经标为「易失分点」。Codex 的答案是**架构性拒绝**：harness 里没有任何意图分类器，而是给模型一个 `request_user_input` 工具（`protocol/src/request_user_input.rs`、`core/src/tools/handlers/request_user_input.rs`），以及 `elicitation.rs`。

**模型自己觉得不确定时，反向问用户**，而不是在 harness 里训练一个分类器。

**这个观点值得展开**：

> 意图分流做在 harness 里，意味着你要维护一套规则/模型，而它永远追不上模型自己的判断力；做在工具层，则是把消歧权交还给最有信息的那个角色。

**代价也要说**：分类器可以离线评测、可以硬编码兜底，而工具层依赖模型愿意承认自己不清楚——这需要 prompt 工程配合。

---

<a id="q10"></a>
## Q10. 模型一致性 & Token 成本

**模型一致性** —— Codex 用**能力声明收窄**，不用 prompt 兼容（`codex-rs/model-provider/src/provider.rs:59`）：

```rust
pub struct ProviderCapabilities {
    pub namespace_tools: bool,
    pub image_generation: bool,
    pub web_search: bool,
    pub external_web_access: bool,
    pub remote_compaction: RemoteCompactionSupport,
}
```

注释写得很清楚：

> These capabilities are a provider-owned upper bound. Callers can disable more functionality through normal config, but should not expose a feature that the active provider marks unsupported here.

即：供应商声明能力上限，配置只能在这个基础上**收窄，不能突破**。这是「用类型系统表达兼容性」而不是「用 prompt 求模型配合」。

**Token 成本** —— `session/token_budget.rs`（`resolve_token_budget` / `apply_model_defaults`）、`session/rollout_budget.rs`（`record_rollout_budget_usage` + `maybe_record_reminder`，超预算主动提醒）、app-server 里的 `turn_cost_worker.rs`（按轮计费上报）。

---

<a id="q11"></a>
## Q11. 其他题目速查表

| 面经题 | Codex 的做法 |
|---|---|
| 写操作边界如何划分 | `AskForApproval`（`UnlessTrusted` / `OnRequest` / `Granular` / `Never`）+ `SandboxPolicy`（`ReadOnly` / `WorkspaceWrite` / `DangerFullAccess` / `ExternalSandbox`）。**二维正交**：审批决定「要不要问人」，沙箱决定「能碰什么」 |
| MCP 的 SSE vs streamable-http | `rmcp-client` 支持四种 transport：`local_stdio_transport` / `streamable_http_retry` / `in_process_transport` / `executor_process_transport`；OAuth 完整（`oauth_client_registration`、`www_authenticate`、`enterprise_oauth_login`） |
| MCP 和 function call 的区别 | 在 Codex 里两者**统一收口**：`build_tool_call` 不区分来源，MCP 工具和内置工具都是 `ToolCall`，走同一个 `ToolRouter`；区别只在 `ToolExposure` 和审批模板（`mcp_tool_approval_templates.rs`） |
| 可观测性怎么做 | `otel` + `call_trace`（Received / result_ready 成对）+ `tool_dispatch_trace` + `rollout-trace` 的 `replay_bundle`（**可回放整个会话**） |
| 为什么选 SQLite | `state` crate + `thread-store`（`LocalThreadStore` / `InMemory`）+ `state_db_bridge`。**但关键设计是 rollout 用 append-only 文件而不是 DB**——本地优先、可 diff、可回放 |
| 5000-6000 人如何扩容 | 不做服务端共享状态：每个用户是自己的进程 + 自己的 `~/.codex`。真要扩容，压力在**模型端**不在 harness |

---

<a id="appendix"></a>
## 附：这份面经本身的可信度

题目来源是牛客、CSDN、知乎、小红书等平台的公开面经贴。需要说明：

**⚠️ 其中有相当一部分是引流卖课内容**（尤其带「小白也能看懂」「收藏版」字样的），里面的「人才缺口 400 万」之类数据不能当真。

**真正有价值的信号是面试官反复追问的方向。**在近十篇面经里，以下五点全都出现过，值得优先准备：

1. Harness 设计
2. 记忆模块
3. 工具容错
4. 上下文管理
5. 成本控制

**另外一个提醒**：面经提到的 Java 八股（HashMap / B 树 / 深分页 / Redis 数据结构）和算法（最长递增子序列、第 k 大）跟本项目无关，但确实是必考项，别因为项目是 Rust 就跳过。

### 来源

- [小红书大模型平台 Agentic 全栈研发练习生 一面凉经 — 牛客](https://www.nowcoder.com/feed/main/detail/e5e9311a623940eead6ec98c65e7f9e8?sourceSSR=subject)
- [小红书 Agent 服务端开发实习一面 — 牛客](https://www.nowcoder.com/discuss/926917172448759808?sourceSSR=dynamic)
- [小红书 AI Agent 一面凉经 — CSDN](https://agent.csdn.net/6a7e8bcc10ee7a33f29ad1bd.html)
- [小红书 Agent 开发一面凉经 — 牛客](https://www.nowcoder.com/feed/main/detail/223fa82f976b4418b214894bd7d941fd?sourceSSR=dynamic#1)
- [小红书大模型二面：在 Agent 中，记忆模块你一般会怎么设计？](https://gitcode.csdn.net/69e38b280a2f6a37c5a0c3da.html#1)
- [小红书三面后端 Agent 方向实习面经](https://mj.mianlingai.com/interview/xiaohongshu-backend-agent-intern-2870290/)
- [小红书二面大模型面试题：Agent 基本架构核心组件详解 — CSDN](https://blog.csdn.net/m0_48891301/article/details/159643849)
- [分享｜4 年经验面小红书 AI 后端，三轮技术面全过 — 牛客](https://www.nowcoder.com/feed/main/detail/102f375fadb749f5803d4d977551fc89?sourceSSR=dynamic#1)
- [小红书 AI 全栈开发二面面经 — 牛客](https://www.nowcoder.com/feed/main/detail/b0aba5ebac994002a27bf0892b8991f9?toCommentId=22759814#1)
- [算法面经分享：小红书智能客服算法 — 知乎](https://zhuanlan.zhihu.com/p/1999882118780696003)
- [一上午面了 6 个 Agent 开发，全是半吊子 — 什么值得买](https://post.smzdm.com/p/az8x0vgr/)
