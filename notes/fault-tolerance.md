# 容错设计

覆盖四块：退避重试、错误分类、panic 传播链、上下文压缩。

## 1. 指数退避 + 抖动

`codex-rs/core/src/util.rs:86-91`：

```rust
pub fn backoff(attempt: u64) -> Duration {
    let exp = BACKOFF_FACTOR.powi(attempt.saturating_sub(1) as i32);
    let base = (INITIAL_DELAY_MS as f64 * exp) as u64;
    let jitter = rand::rng().random_range(0.9..1.1);
    Duration::from_millis((base as f64 * jitter) as u64)
}
```

常量（`util.rs:6-7`）：

```rust
const INITIAL_DELAY_MS: u64 = 200;
const BACKOFF_FACTOR: f64 = 2.0;
```

即 `200ms × 2^(n-1) × jitter(0.9..1.1)`。

**抖动是加分点**：裸指数退避会造成重试风暴同步（thundering herd）——所有客户端在同一时刻撞上服务端。±10% 抖动把尖峰打散。

## 2. 优先尊重服务端信号

`codex-rs/core/src/responses_retry.rs:112`：

```rust
let delay = err.retry_delay().unwrap_or_else(|| backoff(retry_count));
```

服务端给了 `retry-after` 就听服务端的，没有才本地退避。这是「客户端退避」与「服务端背压」的正确优先级。

## 3. 传输降级

`codex-rs/core/src/responses_retry.rs:92-107`：

```rust
if retry_state.retries >= max_retries
    && client_session.try_switch_fallback_transport(
        &turn_context.session_telemetry,
        turn_context.model_info(),
    )
{
    sess.send_event(
        turn_context,
        EventMsg::Warning(WarningEvent {
            message: format!("Falling back from WebSockets to HTTPS transport. {err:#}"),
        }),
    )
    .await;
    retry_state.retries = 0;
    return Ok(());
}
```

WebSocket 连续失败到达上限后，自动降级到 HTTPS，**并且重置重试计数**（`retries = 0`）——换传输等于换了失败面，之前的失败不该计入新通道的配额。

## 4. 错误分类

`codex-rs/protocol/src/error.rs:372` 的 `is_retryable()` 是穷尽匹配。

**不可重试**：

```rust
CodexErrorDetails::TurnAborted
| CodexErrorDetails::SessionBudgetExceeded
| CodexErrorDetails::Interrupted
| CodexErrorDetails::EnvVar(_)
| CodexErrorDetails::Fatal(_)
| CodexErrorDetails::UsageNotIncluded
| CodexErrorDetails::QuotaExceeded
| CodexErrorDetails::InvalidImageRequest()
| CodexErrorDetails::InvalidRequest(_)
| CodexErrorDetails::ToolCollision(_)
| CodexErrorDetails::RefreshTokenFailed(_)
| CodexErrorDetails::UnsupportedOperation(_)
| CodexErrorDetails::Sandbox(_)
| CodexErrorDetails::LandlockSandboxExecutableNotProvided
| CodexErrorDetails::RetryLimit(_)
| CodexErrorDetails::ContextWindowExceeded
| CodexErrorDetails::ThreadNotFound(_)
| CodexErrorDetails::AgentLimitReached { .. }
| CodexErrorDetails::Spawn
| CodexErrorDetails::SessionConfiguredNotFirstEvent
| CodexErrorDetails::UsageLimitReached(_)
| CodexErrorDetails::ServerOverloaded
| CodexErrorDetails::CyberPolicy { .. }
| CodexErrorDetails::MisalignmentPolicyViolation { .. } => false,
```

**可重试**：

```rust
CodexErrorDetails::Stream(..)
| CodexErrorDetails::RateLimitExceeded(_)
| CodexErrorDetails::Timeout
| CodexErrorDetails::RequestTimeout
| CodexErrorDetails::UnexpectedStatus(_)
| CodexErrorDetails::ResponseStreamFailed(_)
| CodexErrorDetails::ConnectionFailed(_)
| CodexErrorDetails::InternalServerError
```

分类原则很清楚：**网络与容量问题重试，语义与策略问题不重试**。重试一个 `InvalidRequest` 只会得到同样的错误。

## 5. 让不让模型参与错误恢复

Codex 的答案是**分类处理**，这是个很好的折中方案范本。

分诊点：`codex-rs/core/src/stream_events_utils.rs:300` 的 `handle_output_item_done`，唯一分类器是 `codex-rs/core/src/tools/router.rs:244` 的 `build_tool_call`：

```rust
pub fn build_tool_call(item: ResponseItem) -> Result<Option<ToolCall>, FunctionCallError> {
```

三种处理：

| 情况 | 处理 | 效果 |
|---|---|---|
| 参数解析失败 | `FunctionCallError::RespondToModel` | 错误文本进历史，**模型自己改** |
| 工具执行失败 | `FunctionCallOutput { success: Some(false) }` | 同上 |
| 工具 task panic | `FunctionCallError::Fatal` | **终止整个 turn** |

### 一个观察到的缺口

`codex-rs/core/src/tools/parallel.rs:242`：

```rust
fn tool_task_join_error(err: JoinError) -> FunctionCallError {
    FunctionCallError::Fatal(format!("tool task failed to receive: {err:?}"))
}
```

把 `JoinError`（包括 panic）一律提升成 `Fatal`，等于放弃了这条路本来最该发挥作用的场景——**工具崩了本可以让模型换个打法重试，现在却直接让整轮对话失败**。`err.is_panic()` 本可以走 `RespondToModel`。

（这是阅读代码得出的观察，不是上游的既定结论。）

## 6. 模型返回 JSON 格式异常怎么处理

分四层：

**① 不预解析** —— `codex-rs/protocol/src/models.rs:1069-1071`，`FunctionCall.arguments` 刻意保持 `String`：

```rust
// The Responses API returns the function call arguments as a *string* that contains
// JSON, not as an already‑parsed object. We keep it as a raw string here and let
// Session::handle_function_call parse it into a Value.
arguments: String,
```

**② 解析失败是可回报的错误，不是崩溃** —— `build_tool_call` 返回 `Result<Option<ToolCall>, FunctionCallError>`，解析失败走 `RespondToModel` 而非 `?`。

**③ 结构化约束优先** —— 工具 schema 走 `ToolSpec`（JSON Schema），不靠 prompt 要求模型「输出 JSON」。

**④ 兜底** —— 完全不认识的 item 落到 `_ => Ok(None)`，当作普通输出收尾。**永远不会因为模型返回了意外类型而崩。**

## 7. 工具的并发门控

`codex-rs/core/src/tools/parallel.rs:109` 和 `158-161`：

```rust
let supports_parallel = router.tool_supports_parallel(&call);
// ...
let guard = if supports_parallel {
    Either::Left(lock.read().await)      // 只读类工具：共享锁，真并行
} else {
    Either::Right(lock.write().await)    // 副作用类工具：独占锁，串行
};
```

**不是全局串行，也不是无脑并行**——用 `RwLock` 按工具是否只读分层。读类工具（搜索、读文件）并发跑，写类工具（执行命令、改文件）互斥。

工具本身跑在自己的 task 里，`parallel.rs:150`：

```rust
let mut dispatch_handle = AbortOnDropHandle::new(tokio::spawn(
```

`AbortOnDropHandle` 意味着 handle 一旦 drop，task 自动取消——不会泄漏。

## 8. 上下文压缩

### 双阈值

`ContextWindowTokenStatus` 同时维护 `auto_compact_scope_limit` 和 `full_context_window_limit`，区分「该开始整理了」和「要撑爆了」——前者软触发，后者硬触发。

### 预判式压缩

`codex-rs/core/src/session/turn.rs:179`：

```rust
// TODO(ccunningham): Pre-turn compaction runs before context updates and the
// new user message are recorded. Estimate pending incoming items (context
// diffs/full reinjection + user input) and trigger compaction preemptively
// when they would push the thread over the compaction threshold.
```

**在拼装 prompt 之前就估算本轮要新增的 token**，提前压——而不是等请求失败再补救。`turn.rs:1244` 有对应的实现说明。

### 不是原地压缩，是「开新窗口」

`should_roll_over` + `take_new_context_window_request()`，压缩产物带 `CompactedHistoryMetadata { window_number, window_ids, compaction_response_id, compaction_model_hash }`。历史是**多窗口可回溯**的结构，不是被就地覆盖。旧窗口还在 rollout 里，可以回放。

### 压缩本身也会失败

独立的 `codex-rs/core/src/compact_model_fallback.rs`——压缩用哪个模型、降级到哪个都有 fallback。

相关文件：`compact.rs`、`compact_token_budget.rs`、`compact_remote_history.rs`、`compact_remote_v2.rs`。

### 一个观察到的取舍

`codex-rs/core/src/session/turn.rs:611`：

```rust
// as long as compaction works well in getting us way below the token limit, we shouldn't worry about being in an infinite loop.
```

**没有强制终止循环的保护**，赌的是压缩能把 token 降得足够低。聊到「死循环 / 成本失控」时这是可以被追问的点。

（同样是阅读代码得出的观察。）

## 9. 没有熔断器

Codex 有重试上限、有传输降级，但**没有 circuit breaker**（半开状态、失败率阈值、快速失败）。

诚实的判断：本地 CLI 场景下熔断价值有限——单用户、单进程，不存在下游雪崩，所以用 `max_retries` + 用户可见的 `Reconnecting... n/N` 提示代替。但要做成多租户服务，这一层必须补。

## 10. 汇总

| 机制 | 有？ | 位置 |
|---|---|---|
| 指数退避 | ✅ | `core/src/util.rs:86` |
| 抖动 | ✅ | 同上，±10% |
| 尊重 `retry-after` | ✅ | `core/src/responses_retry.rs:112` |
| 传输降级 | ✅ | `core/src/responses_retry.rs:92-107` |
| 错误分类 | ✅ | `protocol/src/error.rs:372` |
| 错误回灌给模型 | ✅ | `FunctionCallError::RespondToModel` |
| 工具级并发门控 | ✅ | `core/src/tools/parallel.rs:158-161` |
| 上下文压缩 | ✅ | `core/src/session/turn.rs:179` |
| 会话可回放 | ✅ | rollout-trace `replay_bundle` |
| 熔断器 | ❌ | — |
| 工具 panic 降级 | ❌ | `parallel.rs:242` 一律 Fatal |
