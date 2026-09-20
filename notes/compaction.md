# 上下文压缩全解

压缩什么时候触发、走哪条实现、历史被替换成什么样、prompt 模板长啥样。简版见 [fault-tolerance.md](fault-tolerance.md) §8，这里展开。

## 1. 先破一个误解：ReAct 每轮不压缩

内层循环每轮都是**全量 append-only 重发**，`codex-rs/core/src/session/turn.rs:512-516`：

```rust
// Construct the input that we will send to the model.
let sampling_request_input: Vec<ResponseItem> = async {
    sess.clone_history()
        .await
        .for_prompt(&step_context.settings.model_info.input_modalities)
}
```

`clone_history()` 拿全部历史拼 prompt。压缩不是每轮的常驻动作，而是**阈值触发的整史替换**——平时历史只增不减。

## 2. 触发点地图

| # | 时机 | 条件 | 位置 |
|---|---|---|---|
| T1 | 回合开始前 | token 上限到了 | `turn.rs:1240-1256` |
| T2 | 回合开始前 | comp-hash 变了（模型/配置换血） | `turn.rs:1330-1340` |
| T3 | 回合开始前 | 模型降档 | `turn.rs:1378-1385` |
| T4 | 回合开始前 | Guardian 预算耗尽 | `turn.rs:344-350` |
| T5 | **回合中间** | rollover 条件满足 | `turn.rs:600-601` |
| T6 | 回合中间 | Guardian 预算耗尽 | `turn.rs:724-741` |
| T7 | 手动 | `/compact` | `codex-rs/core/src/session/handlers.rs:243` |

T1 的注释写明了预判式设计（`turn.rs:1244-1245`）：

```rust
// Pre-turn compaction runs before run_turn creates the normal sampling step.
```

进回合之前先看 token 状态，压完再开始正常采样——不是等请求失败再补救。

### T5：回合中间的 rollover

```rust
let should_roll_over = needs_follow_up
    && (sess.take_new_context_window_request().await || token_limit_reached);
```

（`turn.rs:600-601`）两个条件：**还要继续转**（模型有后续或用户插话），且（模型主动要了新窗口 `new_context` 工具，或 token 到顶）。`take_` 前缀 = 取走即清的一次性标志。

紧跟着的注释是个坦诚的赌注（:611）：

```rust
// as long as compaction works well in getting us way below the token limit, we shouldn't worry about being in an infinite loop.
```

压缩失败循环没有强制终止保护（[fault-tolerance.md](fault-tolerance.md) §8 已记）。

满足 rollover 后（:612-639）：跑压缩 → `can_drain_pending_input = !model_needs_follow_up`（:638）→ `continue`（:639）。压缩成功就接着转，回合不死。

## 3. 三条实现路径

`run_auto_compact`（`turn.rs:1397-1462`）是个三路分发：

```rust
if turn_context.config.features.enabled(Feature::TokenBudget) {
    // Compaction is the reset request, so force a new context window
    // instead of consuming a pending `new_context` tool request.
    crate::compact_token_budget::run_inline_auto_compact_task(
        Arc::clone(sess),
        step_context,
        initial_context_injection,
    )
    .await?;
    return Ok(());
}

match turn_context.provider.capabilities().remote_compaction {
    RemoteCompactionSupport::V2 => {
        // ……远端压缩
        run_inline_remote_auto_compact_task_v2(/* …… */).await?;
    }
    RemoteCompactionSupport::Unsupported => {
        // ……本地压缩
        run_inline_auto_compact_task(/* …… */).await?;
    }
}
```

| 路径 | 机制 | 默认？ |
|---|---|---|
| TokenBudget 特性 | 直接重置上下文窗口，**不调 LLM** | ❌ `default_enabled: false`（`codex-rs/features/src/lib.rs:1615-1618`，`Stage::UnderDevelopment`） |
| Remote v2 | 服务端压缩 | 看模型能力 |
| 本地摘要 | 额外调一次 LLM 总结 | ✅ |

所以**当前默认路径是：让 AI 自己压缩**——多发一次 LLM 调用，产出摘要，替换整史。

## 4. 阈值：90% 软线，95% 硬线

`codex-rs/protocol/src/openai_models.rs:521-531`：

```rust
pub fn auto_compact_token_limit(&self) -> Option<i64> {
    let context_limit = self
        .resolved_context_window()
        .map(|context_window| (context_window * 9) / 10);
    let config_limit = self.auto_compact_token_limit;
    if let Some(context_limit) = context_limit {
        return Some(
            config_limit.map_or(context_limit, |limit| std::cmp::min(limit, context_limit)),
        );
    }
    config_limit
}
```

软线 = min(用户配置, 窗口 × 9/10)——**用户只能配得更小，配不过 90%**。硬线是 `usable_context_window`（:513-517）：

```rust
pub fn usable_context_window(&self) -> Option<i64> {
    self.resolved_context_window().map(|context_window| {
        context_window.saturating_mul(self.effective_context_window_percent) / 100
    })
}
```

`effective_context_window_percent` 默认 95（:389-391）。90% 触发压缩，95% 是请求能带的上限——中间 5% 是压缩本身的操作空间。

## 5. 本地压缩算法

一次额外的 LLM 调用，`codex-rs/core/src/compact.rs:236`（`run_compact_task_inner_impl`）：

1. **克隆历史 + 追加压缩指令**（:262-266）：模板作为一条 user 输入塞到历史末尾（`input` 参数），整史发给模型
2. **取摘要**（:349-354）：拿这次调用的最后一条 assistant 消息当摘要，套上前缀——

```rust
let summary_suffix =
    get_last_assistant_message_from_turn(history_snapshot.raw_items()).unwrap_or_default();
let summary_text = format!("{SUMMARY_PREFIX}\n{summary_suffix}");
```

3. **挑幸存者**（:359 → :568-587）：只留 `TurnItem::UserMessage`，assistant/工具调用/工具输出全部丢弃；旧摘要也丢（:576 的 `is_summary_message` 检查）——**防套娃，没有摘要的摘要**

```rust
let Some(TurnItem::UserMessage(user)) = crate::event_mapping::parse_turn_item(item) else {
    return None;
};
if is_summary_message(&user.message()) {
    return None;
}
```

4. **20k token 预算，新→旧装填**（:678-720）：`user_messages.iter().rev()` 从最新的往回装，装不下的截断：

```rust
let mut remaining = max_tokens;
for message in user_messages.iter().rev() {
    if remaining == 0 {
        break;
    }
    let tokens = approx_token_count(&message.message);
    if tokens <= remaining {
        selected_messages.push(message.clone());
        remaining = remaining.saturating_sub(tokens);
    } else {
        let truncated =
            truncate_text(&message.message, TruncationPolicy::Tokens(remaining));
```

`max_tokens` = `COMPACT_USER_MESSAGE_MAX_TOKENS: usize = 20_000`（:61）。

5. **摘要以 user 身份垫底**（:712-719, :744-751）：幸存用户消息依次入史（`role: "user"`），摘要**最后**追加——空摘要也有 `"(no summary available)"` 兜底

6. **整史替换 + 重算 token**（:382-393）：`replace_compacted_history` 换掉全部历史，`recompute_token_usage` 防止刚压完又触发

7. **给用户一条警告**（:399-403）：

```rust
let warning = EventMsg::Warning(WarningEvent {
    message: "Heads up: Long threads and multiple compactions can cause the model to be less accurate. Start a new thread when possible to keep threads small and targeted.".to_string(),
});
```

### 为什么摘要必须是最后一条

`compact.rs:62-77` 的 `InitialContextInjection` 文档注释：

> Mid-turn compaction must use `BeforeLastUserMessage` because **the model is trained to see the compaction summary as the last item in history** after mid-turn compaction

压缩产物的布局是模型训练时的既定格式，摘要垫底不能乱动。

## 6. 模板全文

压缩指令 `codex-rs/prompts/templates/compact/prompt.md`（全文）：

```markdown
You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task.

Include:
- Current progress and key decisions made
- Important context, constraints, or user preferences
- What remains to be done (clear next steps)
- Any critical data, examples, or references needed to continue

Be concise, structured, and focused on helping the next LLM seamlessly continue the work.
```

9 行，没有 JSON schema、没有固定小节，结构让模型自己定。摘要前缀 `summary_prefix.md`（全文）：

```markdown
Another language model started to solve this problem and produced a summary of its thinking process. You also have access to the state of the tools that were used by that language model. Use this to build on the work that has already been done and avoid duplicating work. Here is the summary produced by the other language model, use the information in this summary to assist with your own analysis:
```

三个要点：

- **读者是下一个 LLM 不是人**：①说 "handoff summary for another LLM"，②以第二人称对下一个模型喊话。模型写给模型的交接文档
- 模板作为**user 消息**追加（`compact.rs:117-131` 的 `UserInput::Text`），不是 system message
- `config.compact_prompt` 可以整个换掉模板（:125 的 `unwrap_or(SUMMARIZATION_PROMPT)`）
- 前缀同时是**防递归标记**：`is_summary_message` 用 `starts_with(SUMMARY_PREFIX)` 识别旧摘要（:593-595），下一轮压缩直接丢弃

## 7. 配套机制

**`new_context` 工具**（`codex-rs/core/src/tools/handlers/new_context_window.rs:38`）：模型可以主动调用，设置一次性标志，被 T5 的 `take_new_context_window_request()` 消费。工具描述直言（:14）：

```rust
"A new context window will start without summarizing conversation history.";
```

——要新窗口但不摘要，跟压缩是两条路。

**硬溢出不压缩**（`turn.rs:1613-1614`）：

```rust
CodexErrorDetails::ContextWindowExceeded => {
    sess.set_total_tokens_full(&turn_context).await;
    return Err(err);
}
```

中途真的撑爆了**不会**当场压缩抢救——直接报错结束回合（`ContextWindowExceeded` 不可重试，见 [fault-tolerance.md](fault-tolerance.md) §4），`set_total_tokens_full` 把状态标记满，给**下一回合的 T1** 上膛。压缩永远发生在「还来得及」的时候。

## 8. Go 翻译（触发骨架）

```go
needsFollowUp := modelNeedsFollowUp || hasPendingInput
tokenLimitReached := tokenStatus.LimitReached

// T5：回合中间的 rollover 判断
shouldRollOver := needsFollowUp && (sess.TakeNewContextWindowRequest() || tokenLimitReached)
if shouldRollOver {
	if err := runAutoCompact(ctx, sess, ReasonContextLimit, PhaseMidTurn); err != nil {
		return err // TurnAborted 往上传，其余结束回合
	}
	canDrainPendingInput = !modelNeedsFollowUp // 压缩后模型若还要继续，先让它继续
	continue
}
if !needsFollowUp {
	break
}
```

`TakeNewContextWindowRequest` 的取走即清：

```go
func (s *Session) TakeNewContextWindowRequest() bool {
	return s.newContextRequested.Swap(false) // atomic take：读的同时清掉
}
```

## 9. 没验证的

- 传输层的 WS-v2 增量 delta（历史层是全量重发，传输层可能只发增量）没在本 fork 树核对行号。
- Remote v2 路径（`compact_remote_v2.rs`）的服务端协议细节没读。
- `replace_compacted_history`（`session/mod.rs:3943`）落盘时的 `CompactedItem` checkpoint 格式没展开。
