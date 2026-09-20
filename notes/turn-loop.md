# 两层循环与审批挂起

回合（turn）怎么跑、用户怎么插话、审批怎么把回合「暂停」住而不结束它。

## 1. 两层循环的分工

```
外层 RegularTask::run        —— 插话入口，决定要不要再进一轮（tasks/regular.rs:75-96）
  └─ run_turn                —— ReAct 主循环（session/turn.rs:423 loop）
       ├─ 拉取 pending input（用户插话）
       ├─ clone_history().for_prompt()   全量拼 prompt（append-only 重发）
       ├─ 采样（LLM 调用）
       ├─ 工具执行
       └─ 判断 needs_follow_up → break / continue / 压缩后 continue
```

外层循环 `codex-rs/core/src/tasks/regular.rs:75-96`：

```rust
let mut next_input = input;
loop {
    let last_agent_message = run_turn(
        Arc::clone(&sess),
        Arc::clone(&ctx),
        next_input,
        &mut mcp_startup_requirements,
        prewarmed_client_session.take(),
        cancellation_token.child_token(),
    )
    .instrument(run_turn_span.clone())
    .await?;
    // Terminal errors are already reported. Let task completion preserve pending
    // input instead of restarting the failed turn for that same input.
    if ctx.terminal_error.lock().await.is_some() {
        return Ok(last_agent_message);
    }
    if !sess.input_queue.has_pending_input(&sess.active_turn).await {
        return Ok(last_agent_message);
    }
    next_input = Vec::new();
}
```

外层退出信号只有两个：`terminal_error`（:89）或 `input_queue` 空（:92）。注意 :95 的 `next_input = Vec::new()`——传给下一轮 run_turn 的**不是数据，是开关**：内层看到 `input.is_empty()` 就自己去 input_queue 拉插话（`turn.rs:425-435` 的 `can_drain_pending_input` 门控）。

内层退出（`turn.rs:640-704`）：

- `!needs_follow_up` → 跑 stop hooks → `break`（:702；stop hook 要求继续则 :690 的 `should_stop` 分支提前 break）
- `needs_follow_up` → `continue`（:704）
- 中途压缩成功 → `continue`（:639），压缩后接着跑

`needs_follow_up = model_needs_follow_up || has_pending_input`（`turn.rs:565`）——模型还要继续（工具没跑完）**或**用户插了话，都算「需要下一轮」。这就是插话不打断模型的实现：插话只是让内层多转一圈。

| 层 | 职责 | 写盘 | 压缩 |
|---|---|---|---|
| 外层 | 判断有没有插话、要不要再进一轮 | ❌ | ❌ |
| 内层 | 采样、工具、历史记录、压缩、全部持久化 | ✅ | ✅ |

## 2. 审批：协作式挂起，不是结束回合

审批到来时**回合不死**，是循环里的一次 `.await` 挂起。`codex-rs/core/src/session/mod.rs:2789` 的 `request_command_approval`：

```rust
// Add the tx_approve callback to the map before sending the request.
let (tx_approve, rx_approve) = oneshot::channel();
let prev_entry = {
    let mut active = self.active_turn.lock().await;
    match active.as_mut() {
        Some(at) => {
            let mut ts = at.turn_state.lock().await;
            ts.insert_pending_approval(effective_approval_id.clone(), tx_approve)
        }
        None => None,
    }
};
```

（:2808-2821）先把 sender 塞进 pending map，再给用户发审批事件（:2874），最后挂起等决议：

```rust
rx_approve.await.unwrap_or(ReviewDecision::Abort)
```

（:2875）

两个刻意的顺序：

1. **先注册再发送**（:2808 注释原话）。反过来写，用户的手速快过注册，响应就永远丢了。
2. **`unwrap_or(Abort)`**：oneshot 的另一半被 drop——即整个 turn 被 interrupt、任务连带 drop——`await` 返回 `None`，视为 Abort。取消路径不需要任何人「通知」这条 await。

唤醒走的是同一条 Op 提交通道。用户答复 → `Op::ExecApproval` → `codex-rs/core/src/session/handlers.rs:200` → `notify_approval`（`session/mod.rs:3364`）：

```rust
let entry = {
    let mut active = self.active_turn.lock().await;
    match active.as_mut() {
        Some(at) => {
            let mut ts = at.turn_state.lock().await;
            ts.remove_pending_approval(approval_id)
        }
        None => None,
    }
};
match entry {
    Some(tx_approve) => {
        tx_approve.send(decision).ok();
    }
    None => {
        warn!("No pending approval found for call_id: {approval_id}");
    }
}
```

（:3365-3381）`remove_pending_approval` 拿走 sender 就地发送。它要求 `active_turn` 存在——turn 都没了自然无审批可唤醒，只剩一条 warn。

前提是 `codex-rs/core/src/session/session.rs:50` 的文档保证：「A session has at most 1 running task at a time」——turn_state 里的 pending map 才不会跨 turn 串号。

## 3. 审批的三级优先级

`codex-rs/core/src/tools/approvals.rs:487-489` 原话：

```rust
// Approval precedence is:
// 1. Hooks
// 2. If StrictAutoReview || Guardian enabled, then Guardian. Else, user.
```

Hooks（:490 的 `run_permission_request_hooks`）决了就不问人；没意见才落到 :512 的 `request_reviewer_approval`——Guardian（自动审查）开着问 Guardian，否则问用户。用户看到的审批框，其实是这条链的最后一环。

## 4. Go 翻译

| Rust | Go | 注意 |
|---|---|---|
| `oneshot::channel()` | `ch := make(chan T, 1)` | 缓冲 1 是关键：发送方永不因接收方先走而卡死 |
| `rx.await.unwrap_or(Abort)` | `select { case v := <-ch: ...; case <-ctx.Done(): return Abort }` | drop ≈ ctx 取消 |
| pending map 的 take 语义 | `v, ok := m[id]; delete(m, id)` 或 `atomic.Value` 的 Swap | 取走即删除，防双发 |
| 无 RAII / Drop | turn 结束显式清理 pending map | Go 没有 Drop guard，漏了就泄漏 |

审批挂起的骨架：

```go
func (s *Session) RequestApproval(ctx context.Context, id string) Decision {
	ch := make(chan Decision, 1) // oneshot：缓冲 1
	s.mu.Lock()
	if s.turnState == nil { // active_turn 不存在 → 无处注册
		s.mu.Unlock()
		return Abort
	}
	s.pending[id] = ch // 先注册
	s.mu.Unlock()

	s.emit(ApprovalRequest{ID: id}) // 再发送
	select {
	case d := <-ch: // notify: delete(m,id) 后 ch <- d
		return d
	case <-ctx.Done(): // turn 被 interrupt ≈ oneshot 另一半 drop
		return Abort
	}
}
```

## 5. 一个观察

「挂起在 turn 内部」意味着审批期间这个 tokio task 一直活着，占着 active_turn 位。好处是上下文（栈上的所有局部状态、client_session）原样保留，审批通过后无缝继续；代价是审批如果永远不来，这个 task 就永远挂着——只能靠 interrupt（drop 整个 task）收场，没有「超时自动 Abort」。（阅读观察，非上游结论。）
