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

三个刻意的顺序：

1. **先注册再发送**（:2808 注释原话）。反过来写，用户的手速快过注册，响应就永远丢了。

2. **`unwrap_or(Abort)`**：sender 被 drop 时 `await` 返回 `None`，视为 Abort。**没有超时机制——「表被清掉」就是超时机制。** sender 的死法有三条：

   | 死法 | 位置 | 触发 |
   |---|---|---|
   | 被新 entry 覆盖 | `state/turn.rs:122-128` `insert_pending_approval` 把旧 sender 当返回值交出去，最后随 `prev_entry` 一起 drop | 同一个 `call_id` 二次请求审批 |
   | 整张表被清 | `state/turn.rs:137-142` `clear_pending_waiters` 一次 `.clear()` 五张表 | turn abort / turn suspension |
   | 表随 turn 一起 drop | `TurnState` 是 `ActiveTurn` 的字段（`state/turn.rs:34`） | 会话结束 |

   五张表在 `state/turn.rs:90-96`：`pending_approvals` / `pending_request_permissions` / `pending_user_input` / `pending_elicitations` / `pending_dynamic_tools`。

3. **先 cancel 再清表**。`tasks/mod.rs:534-535` 注释原话：

   ```rust
   // Let interrupted tasks observe cancellation before dropping pending approvals, or an
   // in-flight approval wait can surface as a model-visible rejection before TurnAborted.
   ```

   即 `handle_task_abort`（`tasks/mod.rs:878-916`：cancel token → `select!` 等 `task.done` 或 `GRACEFULL_INTERRUPTION_TIMEOUT_MS = 100ms`（:69）→ `task.handle.abort()`）先跑完，`clear_pending` 才执行。两个调用点都在 `tasks/mod.rs`：`:536` 的 `abort_all_tasks`（:512）和 `:582` 的 `abort_turn_if_active`（:543）。顺序倒了，挂着的审批会先以「被拒绝」的错误回灌进对话历史，污染 `TurnAborted` 之前的事件序列。

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

注意 `tx_approve.send()` 是在**块作用域之外**执行的（`entry` 先把 sender 摘出来，两把锁都释放了才 send）。在 Rust 里这是好习惯（wake 不会同步跑接收方代码）；在 Go 里这是**硬约束**——持锁时做阻塞发送可能死锁。

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
| `rx.await.unwrap_or(Abort)` | `d, ok := <-ch; if !ok { return Abort }` | **单出口**。Go 的 `chan` 没有「发送方消失」这个信号，只有 `close` 能产生 `ok == false` |
| sender 被 drop（`Err(RecvError)`） | `close(ch)` | **最容易漏的一处语义差**：Go 不会因为 sender 被 GC 掉而唤醒接收方，必须显式 close |
| interrupt / 取消 | 另加 `case <-ctx.Done():` | ⚠️ **这不是 Rust 的语义**。Rust 那边 `rx_approve.await` 并没有和 cancellation token 做 select（`session/mod.rs:2875`），它靠的是上面第 3 条的**发送侧顺序**，不是接收侧 select |
| pending map 的 take 语义 | `v, ok := m[id]; delete(m, id)` | 取走即删除，防双发 |
| 无 RAII / Drop | turn 结束显式清理 pending map | Go 没有 Drop guard，漏了就泄漏 |

两个必须记住的陷阱：

- **绝不要写 `default:` 分支。** 那会把「没等到答复」变成「直接放行」，审批框就成了装饰。
- **绝不用 `len(ch) == 0` 判空。** 那是把挂起退回成轮询；而且 close 之后 `len` 也是 0，无法区分「还没答复」和「发送方死了」。

审批挂起的骨架：

```go
// 接收侧：单出口，没有 select。
func (s *Session) RequestApproval(id string) Decision {
	ch := make(chan Decision, 1) // oneshot：缓冲 1，发送方永不阻塞
	s.mu.Lock()
	if s.turnState == nil { // active_turn 不存在 → 无处注册
		s.mu.Unlock()
		return Abort
	}
	s.pending[id] = ch // 先注册
	s.mu.Unlock()

	s.emit(ApprovalRequest{ID: id}) // 再发送

	d, ok := <-ch // 阻塞在这里。这不是 bug，是挂起。
	if !ok {
		// 通道被 close == Rust 的 Err(RecvError) == sender 死了。
		// 三种死法（被覆盖 / 被清表 / 表整体 drop）都收敛到这一个分支。
		return Abort
	}
	return d
}

// 发送侧：先摘表，再投递，最后 close。
func (s *Session) NotifyApproval(id string, d Decision) {
	s.mu.Lock()
	ch, ok := s.pending[id]
	if ok {
		delete(s.pending, id) // 先摘表 → 第二次调用拿不到，天然防双发
	}
	s.mu.Unlock() // ★ 必须解锁后再发：Go 持锁时阻塞发送可能死锁
	if !ok {
		return // 对应 warn!("No pending approval found for call_id")
	}
	ch <- d
	close(ch)
}

// 对应 clear_pending_waiters：一次清五张表 = 一次 close 掉五个 waiter。
func (s *Session) ClearPendingWaiters() {
	s.mu.Lock()
	old := s.pending
	s.pending = map[string]chan Decision{}
	s.mu.Unlock()
	for _, ch := range old {
		close(ch) // ★ Go 里「sender 消失」必须显式 close，否则接收方永远醒不来
	}
}
```

## 5. 挂起怎么收场

「挂起在 turn 内部」意味着审批期间这个 tokio task 一直活着，占着 active_turn 位。好处是上下文（栈上的所有局部状态、client_session）原样保留，审批通过后无缝继续；代价是审批如果永远不来，这个 task 就永远挂着——**没有「超时自动 Abort」**。

退场机制不是超时，是**表被清**：`clear_pending_waiters`（`state/turn.rs:137-142`）一次 close 五张表，挂在里面的每个 waiter 一起以 `Abort` 惊醒。触发点是 `session/input_queue.rs:207-211` 的 `clear_pending`，调用方两处：

- `tasks/mod.rs:536`（`abort_all_tasks`）/ `:582`（`abort_turn_if_active`）——turn abort
- `session/turn_suspension.rs:98`——turn 挂起移交，注释原话：「Pending accepted input and interactive waiters live only in this process. Handoff intentionally drops that state; persisting or replaying it needs a separate protocol.」

**一个反直觉的点**：hard abort 路径下，`handle_task_abort` 里的 `task.handle.abort()`（`tasks/mod.rs:916`）已经把 future 丢掉了，所以那个 `rx_approve.await` **根本不会再跑**——它返回的 `None` 没人读。`clear_pending` 排在这之后（:533-536 的顺序），真正的作用对象是**优雅收场**和**仍然活着的其他 waiter**（权限请求 / 用户提问 / elicitation），不是「通知这条 await」。

另外：`request_command_approval` 里的 `rx_approve.await` 并没有和 cancellation token 做 select（`session/mod.rs:2875`），所以一个正停在这次 await 上的 task，理论上在那 100ms 优雅窗口里也感知不到取消。`tasks/mod.rs:534-535` 注释说的「let interrupted tasks observe cancellation」具体靠哪个 await 点生效，没追。

（以上均为阅读观察，非上游结论。）
