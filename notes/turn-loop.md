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

`run_turn(...).await?` 若返回错误，外层会直接传播错误；成功返回后有两个提前结束条件：存在 `terminal_error`（:89-90）或 `input_queue` 没有 pending input（:92-93）。注意 :95 的 `next_input = Vec::new()`——传给下一轮 run_turn 的**不是数据，是开关**：内层看到 `input.is_empty()` 就自己去 input_queue 拉插话（`turn.rs:365, 427-435` 的 `can_drain_pending_input` 门控）。

内层退出/续跑（`turn.rs:600-704`）：

- `needs_follow_up` 为真且压缩未接管时 → `continue`（:704）；中途压缩成功也会 `continue`（:639），并按 `model_needs_follow_up` 决定是否暂缓 drain 插话。
- `!needs_follow_up` → 跑 stop hooks（:642-650）。若 hook 要求 block 且提供 continuation prompt，记录 prompt 后 `continue`（:661-678）；block 却没有 prompt 时只发 warning 并忽略 block（:679-687）。随后 `should_stop` 为真则 `break`（:689-690）；其余成功路径在 :702 `break`。

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

（:2809-2820）先把 sender 塞进 pending map，再发审批事件（:2855-2874），最后挂起等决议：

```rust
rx_approve.await.unwrap_or(ReviewDecision::Abort)
```

（:2875）

三个刻意的顺序：

1. **先注册再发送**（:2808 注释原话）。反过来写，用户的手速快过注册，响应就永远丢了。

2. **`unwrap_or(Abort)`**：这个 command-approval sender 被 drop 时，oneshot `await` 返回 `Err`，于是结果按 `ReviewDecision::Abort` 处理。这里没有审批超时；清理 waiter 表是 turn 退场清理，不是超时机制。对这条审批 waiter，sender 可能因以下原因被 drop：

   | 死法 | 位置 | 触发 |
   |---|---|---|
   | 被新 entry 覆盖 | `state/turn.rs:122-128` `insert_pending_approval` 把旧 sender 当返回值交出去，最后随 `prev_entry` 一起 drop | 同一个 `call_id` 二次请求审批 |
   | 整张表被清 | `state/turn.rs:137-143` `clear_pending_waiters` 清五张 waiter map | turn abort / turn suspension |
   | `TurnState` 随 `ActiveTurn` drop | `state/turn.rs:31-35` | owning active-turn state 被销毁 |

   五张表在 `state/turn.rs:90-96`：`pending_approvals` / `pending_request_permissions` / `pending_user_input` / `pending_elicitations` / `pending_dynamic_tools`。

3. **取消/终止处理先于清表**。`tasks/mod.rs:534-535` 的原注释是：

   ```rust
   // Let interrupted tasks observe cancellation before dropping pending approvals, or an
   // in-flight approval wait can surface as a model-visible rejection before TurnAborted.
   ```

   两个 abort 调用点（`:512-541`、`:543-588`）先进入 `handle_task_abort`（`:878-920`）：取消 token，等待 `task.done` 或最多 `GRACEFULL_INTERRUPTION_TIMEOUT_MS = 100ms`（:69），然后 abort task handle 并调用 task 的 abort 清理。之后由 abort 流程发终态/生命周期事件，再在 `clear_pending`（`input_queue.rs:206-211`）清 waiter 与 pending input；清理具体发生在 `:536` / `:582`。因此不是「清表先于 TurnAborted」：源码注释说明的是先给协作式取消留窗口，避免审批等待被提前变成模型可见拒绝。若超时后硬 abort，审批 future 已被丢弃，清表不再唤醒它。

唤醒走的是同一条 Op 提交通道。用户答复经 `Op::ExecApproval` 在 submission loop 分派（`session/handlers.rs:529-535`）；`exec_approval` 对非 `Abort` 决议在 `:196-200` 调 `notify_approval`，而 `Abort` 会走 `interrupt_task()`（:197-199）。`notify_approval` 位于 `session/mod.rs:3364-3383`：

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

注意 `tx_approve.send()` 是在**块作用域之外**执行的（`entry` 先把 sender 摘出来，两把锁都释放了才 send）。在 Rust 里这样缩短锁作用域；在 Go 里则尤其重要——持锁时做可能阻塞的发送会造成死锁。

前提是 `codex-rs/core/src/session/session.rs:50` 的文档保证：「A session has at most 1 running task at a time」——turn_state 里的 pending map 才不会跨 turn 串号。

## 3. 审批的三级优先级

`codex-rs/core/src/tools/approvals.rs:493-495` 原话：

```rust
// Approval precedence is:
// 1. Hooks
// 2. If StrictAutoReview || Guardian enabled, then Guardian. Else, user.
```

Hooks（:496-512 的 `run_permission_request_hooks`）决了就不问 reviewer；没意见才落到 `request_reviewer_approval`（:542-557）——先尝试 Guardian，若无 Guardian 决议则问用户。用户看到的审批框是这条链的最后一环。

## 4. Go 翻译

| Rust | Go | 注意 |
|---|---|---|
| `oneshot::channel()` | `ch := make(chan T, 1)` | 缓冲 1 可让通知方不依赖接收方此刻是否正在读 |
| `rx.await.unwrap_or(Abort)` | `d := <-ch` | 单次结果通道；清理路径通过投递显式 `Abort` 收尾 |
| sender 被 drop（`Err(RecvError)`） | pending map 摘除后发送终止结果 | Go 不会因 sender 被 GC 而关闭 channel；必须由拥有 waiter 的路径显式发送 |
| interrupt / 取消 | 另设显式 interrupt 路径 | ⚠️ Rust 的 `request_command_approval` 没有将 `rx_approve.await` 与 cancellation token 做 select（`session/mod.rs:2875`）；客户端选 `Abort` 时 `handlers.rs:196-199` 走 `interrupt_task()`，不是普通审批通知 |
| pending map 的 take 语义 | `v, ok := m[id]; delete(m, id)` | 取走即删除，防双发 |
| 无 RAII / Drop | turn 结束显式清理 pending map | Go 没有 Drop guard，漏了就泄漏 |

两个必须记住的陷阱：

- **绝不要写 `default:` 分支。** 那会把「没等到答复」变成「直接放行」，审批框就成了装饰。
- **绝不用 `len(ch) == 0` 判空。** 那是把挂起退回成轮询；是否收到结果应由一次接收决定，而不是通过缓冲长度推断。

审批挂起的骨架：

```go
// Go channel 不会因 sender 被 drop 自动关闭；用 Once 让通知/清理只完成一次。
type ApprovalWaiter struct {
	ch   chan Decision
	once sync.Once
}

func (w *ApprovalWaiter) Resolve(d Decision) {
	w.once.Do(func() {
		w.ch <- d // 缓冲 1，只允许一个终止结果
		close(w.ch)
	})
}

// 接收侧：单出口，没有 select。
func (s *Session) RequestApproval(id string) Decision {
	waiter := &ApprovalWaiter{ch: make(chan Decision, 1)}
	s.mu.Lock()
	if s.turnState == nil { // active_turn 不存在 → 无处注册
		s.mu.Unlock()
		return Abort
	}
	old := s.pending[id]
	s.pending[id] = waiter // 先注册
	s.mu.Unlock()
	if old != nil {
		old.Resolve(Abort) // 覆盖旧 waiter，显式投递终止结果
	}

	s.emit(ApprovalRequest{ID: id}) // 再发送

	d := <-waiter.ch // 阻塞在这里。这不是 bug，是挂起。
	return d
}

// 发送侧：先摘表，再投递一次决议唤醒接收端。
func (s *Session) NotifyApproval(id string, d Decision) {
	if d == Abort {
		s.interruptTask() // Rust handlers.rs:196-199 将 Abort 映射为 interrupt_task
		return
	}
	s.mu.Lock()
	waiter, ok := s.pending[id]
	if ok {
		delete(s.pending, id) // 先摘表 → 第二次调用拿不到，天然防双发
	}
	s.mu.Unlock() // ★ 必须解锁后再发：Go 持锁时阻塞发送可能死锁
	if !ok {
		return // 对应 warn!("No pending approval found for call_id")
	}
	waiter.Resolve(d)
}

// 清理入口的 Go 示例只模拟审批 waiter map（不是全部五张 Rust 表）。
func (s *Session) ClearPendingApprovals() {
	s.mu.Lock()
	old := s.pending
	s.pending = map[string]*ApprovalWaiter{}
	s.mu.Unlock()
	for _, waiter := range old {
		waiter.Resolve(Abort) // 显式结束 waiter；Once 防止与通知重复发送
	}
}
```

## 5. 挂起怎么收场

「挂起在 turn 内部」意味着审批期间这个 tokio task 仍存活，占着 active_turn 位。好处是局部状态、`client_session` 原样保留，审批通过后可继续；若 turn 仍保持 active 且没有批准/中断，审批等待没有超时，会一直挂起。

退场时除了 task 的协作取消/硬 abort，还会清理 waiter 表；`clear_pending_waiters` 本身不是超时器：`state/turn.rs:137-143` 清空五张 map 并 drop 尚存 sender；各 waiter 的接收端对 oneshot 关闭的处理不同，不能概括成「五类 waiter 都以 Abort 醒来」。命令审批的 oneshot 才通过 `unwrap_or(ReviewDecision::Abort)` 映射成 Abort。清理入口是 `session/input_queue.rs:206-211` 的 `clear_pending`，调用方包括：

- `tasks/mod.rs:536`（`abort_all_tasks`）/ `:582`（`abort_turn_if_active`）——turn abort
- `session/turn_suspension.rs:98`——turn 挂起移交，注释原话：「Pending accepted input and interactive waiters live only in this process. Handoff intentionally drops that state; persisting or replaying it needs a separate protocol.」

**一个反直觉的点**：hard abort 路径下，`handle_task_abort` 里的 `task.handle.abort()`（`tasks/mod.rs:916`）已经把 future 丢掉了，所以那个 `rx_approve.await` **根本不会再跑**——它返回的 `None` 没人读。`clear_pending` 排在这之后（:533-536 的顺序），真正的作用对象是**优雅收场**和**仍然活着的其他 waiter**（权限请求 / 用户提问 / elicitation），不是「通知这条 await」。

另外：`request_command_approval` 里的 `rx_approve.await` 并没有和 cancellation token 做 select（`session/mod.rs:2875`）。但需区分整个工具链：审批 reviewer 的上游调用通常可在 `tools/approvals.rs:667-718` 与 `tools/parallel.rs:202-205` 的 cancellation select 响应取消；本段 `rx_approve.await` 本身不能直接响应 cancellation token。

（以上均为阅读观察，非上游结论。）
