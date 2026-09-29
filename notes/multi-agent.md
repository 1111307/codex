# 多智能体架构

一句话：**不是为每个 agent 启动独立 OS 进程，而是在同一进程内为每个 agent 建立独立 Session/异步任务**（Go 心智模型：goroutine + 消息传递）。父和子共用 AgentControl，但会话、回合、历史和信箱各自独立。贯通双层 loop 与沙箱的时序见 [loops-agents-sandbox-go.md](loops-agents-sandbox-go.md)。

## 0. Rust/Tokio → Go 对照表

| Codex (Rust/Tokio) | Go 里的对应物 | 差异要点 |
|---|---|---|
| `tokio::spawn(fut)` | `go f()` | 都是调度并发任务，但 panic、取消和 Join 语义不同 |
| `async_channel::bounded(n)` | `make(chan T, n)` | 都有背压 |
| `async_channel::unbounded()` | 无标准库直接对应 | `chan` 总是无缓冲或固定容量，须自行设计背压/队列 |
| `Arc<Session>` | `*Session` | Go 有 GC，不用手动 Arc；共享可变状态仍要同步 |
| `Mutex<HashMap>` | `sync.Mutex` + `map` | 读写都要保护 |
| `AtomicUsize` + `compare_exchange_weak` | `atomic.Int64.CompareAndSwap` | 配额计数可用 CAS |
| `watch::Receiver<T>` | **没有直接对应** | 要自己封「最新值 + 广播」 |
| `CancellationToken` | `context.Context` | 两者都须在任务中协作检查 |
| `Drop` / `AbortOnDropHandle` | `defer` + 显式 cancel | `defer` 本身不会中止 goroutine |
| `Weak<ThreadManagerState>` | **没有可靠的标准库等价物** | 设计所有权/主动解除引用；`runtime.SetFinalizer` 不是弱引用 |
| `JoinHandle` | 显式 done channel / `sync.WaitGroup` | goroutine 不自带可等待句柄 |

## 1. 三层结构

```
ThreadManagerState        全局注册表，一进程一份
  map[ThreadID]*Agent + rollout store + graph store
        ↑ Weak 引用（打破循环）
AgentControl              一棵 agent 树共享一份
  registry: map[path]*AgentMeta, totalCount, limiter
        ↑ Clone 给每个 agent
Agent (Session/task) ×N  每个 = 独立 Session + 独立 rollout + 独立信箱
```

`codex-rs/core/src/agent/control.rs:117-140`：

```rust
/// Control-plane handle for multi-agent operations.
/// `AgentControl` is held by each session (via `SessionServices`). It provides capability to
/// spawn new agents and the inter-agent communication layer.
/// An `AgentControl` instance is intended to be created at most once per root thread/session
/// tree. That same `AgentControl` is then shared with every sub-agent spawned from that root,
/// which keeps the registry scoped to that root thread rather than the entire `ThreadManager`.
```

**每棵 root session 树共用一份 AgentControl**，spawn 配额按这棵树计算，不是 ThreadManager 的全局计数；不能简单用 UI 窗口数量推断 root 树数量。

`manager: Weak<ThreadManagerState>` 是弱引用，注释原文（`control.rs:127-130`）说明要避免引用环：

```rust
/// This is `Weak` to avoid reference cycles and shadow persistence of the form
/// `ThreadManagerState -> CodexThread -> Session -> SessionServices -> ThreadManagerState`.
```

## 2. 一个 agent = 一个 Session + 收件 task + 回合 task

SQ/EQ 协议（Submission Queue / Event Queue）的物理实现，`codex-rs/core/src/session/mod.rs:574-575`：

```rust
let (tx_sub, rx_sub) = async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY);
let (tx_event, rx_event) = async_channel::unbounded();
```

**一有一无是刻意的**：入站有界，防止调用方灌爆；出站无界，使事件发送不因接收端慢而受固定容量限制，但也意味着内存不能靠 channel 容量封顶。

Go 版必须为事件出口另定队列与背压策略；标准 `chan Event` 不是无界通道：

```go
type SessionIO struct {
    sub  chan Op       // 入口：make(chan Op, n)，固定容量
    done chan struct{} // Session 收件循环退出时显式 close
    // 事件出口需自行选择队列/背压策略，不能把 chan Event 当成 unbounded。
}
```

收件循环，对应 `codex-rs/core/src/session/mod.rs:877-890` 的 `tokio::spawn(submission_loop(...))`：

```rust
// This task will run until Op::Shutdown is received.
let session_for_loop = Arc::clone(&session);
let session_loop_handle = tokio::spawn(async move {
    submission_loop(session_for_loop, configured_config, rx_sub)
        .instrument(info_span!("session_loop", thread_id = %thread_id))
        .await;
});
```

Go 结构示意（只表现收件与回合分离；单活跃 turn 的同步仍需实现）：

```go
func (a *Agent) submissionLoop() {
    defer close(a.IO.done)
    for op := range a.IO.sub {
        switch op.Kind {
        case OpUserInput:
            a.startOrSteerTurn(op) // 无活跃回合时另起 goroutine；有则入 pending 队列
        case OpInterAgentCommunication:
            a.HandlePeerMessage(op)
        case OpExecApproval:
            a.NotifyApproval(op)
        case OpInterrupt:
            a.CancelTurn()
        case OpShutdown:
            return
        }
    }
}
```

**父子运行同一套 Session 回合机制**，但 submission_loop 不能同步调用 `runTurn`：否则等待模型或审批时就收不到审批答复和用户插话。原代码在 `session/turn_input.rs:269-329` 尝试 steer 或 `spawn_task(RegularTask::new())`，后者由 `tasks/mod.rs:360-409` 起单独任务。Go 版收件循环退出时还须取消并等待活跃回合任务；上面仅展示消息分派。

## 3. spawn_agent 全流程

模型调 `spawn_agent` 时，它就是个普通 tool handler：`codex-rs/core/src/tools/handlers/multi_agents/spawn.rs:47-134` 的 `handle_spawn_agent` 构造子 agent 配置，调用 `AgentControl::spawn_agent_with_metadata`；内部进入 `agent/control/spawn.rs:614-819` 的 `spawn_agent_internal`。配置从当前 turn 刷新模型、审批、cwd 和 permission profile（`tools/handlers/multi_agents_common.rs:170-264`），role 在此基础上覆盖允许的字段；`fork_context=true` 才会选择完整父历史 fork（`spawn.rs:94-109, 121-124`）。**这里只建会话和提交输入，尚未启动 OS 命令进程或沙箱。**

深度检查在 handler 里，`multi_agents/spawn.rs:68-74`：

```rust
let child_depth = next_thread_spawn_depth(&session_source);
let max_depth = turn.config.agent_max_depth;
if exceeds_thread_spawn_depth_limit(child_depth, max_depth) {
    return Err(FunctionCallError::RespondToModel(
        "Agent depth limit reached. Solve the task yourself.".to_string(),
    ));
}
```

注意是 `RespondToModel` 而不是 fatal —— **超深度不是崩溃，是把错误回给模型让它自己想办法**。

`exceeds_thread_spawn_depth_limit` 在 `codex-rs/core/src/agent/registry.rs:91-93`：

```rust
pub(crate) fn exceeds_thread_spawn_depth_limit(depth: i32, max_depth: i32) -> bool {
    depth > max_depth
}
```

Go 版全流程（省略 V2 residency slot 与 execution limiter，不能直接运行）：

```go
func (c *AgentControl) SpawnAgent(cfg Config, input []UserInput, src SessionSource) (*LiveAgent, error) {
    // cfg 假设已从父 turn 的生效快照生成；fork_context 分支省略。
    // ① 深度闸门 —— multi_agents/spawn.rs:68-74
    childDepth := src.Depth + 1
    if childDepth > cfg.AgentMaxDepth {
        return nil, ErrRespondToModel("Agent depth limit reached. Solve the task yourself.")
    }

    // ② 抢名额 —— registry.rs:96-115
    slot, err := c.reserveSpawnSlot(c.maxThreads)
    if err != nil {
        return nil, err      // AgentLimitReached
    }
    committed := false
    defer func() {
        if !committed {
            slot.Rollback()  // ← 对应 Rust 的 Drop (registry.rs:393-402)
        }
    }()

    // ③ 建会话并启动收件循环；等到 SessionConfigured 后登记 ThreadManager
    agent, err := c.manager.NewThread(cfg, c, src) // 内部启动并等到 SessionConfigured
    if err != nil { return nil, err }

    // ④ 登记进 registry；commit 后失败必须显式 shutdown/release
    meta := AgentMeta{ID: agent.ID, Path: agent.Path}
    slot.Commit(meta)
    committed = true

    // ⑤ 派首条输入，启动/steer 子 agent 的回合 —— spawn.rs:784-788
    if err := agent.StartOrSteer(input); err != nil {
        agent.ShutdownAndRelease() // Go 设计：commit 后失败显式回收
        return nil, err
    }

    return &LiveAgent{ThreadID: agent.ID, Status: agent.Status()}, nil
}
```

**`slot + committed` 这个模式是 Go 里必须显式写的**。Rust 的 `SpawnReservation` 在 `Drop` 里检查 `self.active`（`agent/registry.rs:393-401`）：

```rust
impl Drop for SpawnReservation {
    fn drop(&mut self) {
        if self.active {
            if let Some(agent_path) = self.reserved_agent_path.take() {
                self.state.release_reserved_agent_path(&agent_path);
            }
            self.state.total_count.fetch_sub(1, Ordering::AcqRel);
        }
    }
}
```

意义是**异常安全的名额与路径占用**：预留 → commit 之前失败 → Drop 自动归还。源码在 commit 后派首条输入（`agent/control/spawn.rs:726-799`）；若此时派发失败，**不能归功于 reservation 的 Drop**，相关生命周期回收须另核对。上面 Go 版的 `ShutdownAndRelease()` 是建议的显式失败清理，不宣称逐行等价于当前 Rust 实现。

## 4. 两道限流闸

| | 限什么 | 什么时候检查 | 释放方式 |
|---|---|---|---|
| `AgentRegistry.total_count` | 活跃 agent **总数** | spawn 时 | `release_spawned_thread` |
| `AgentExecutionLimiter` | **同时在跑 turn** 的 agent 数 | turn 开始时 | RAII guard，`Drop` 自动释放 |

第二道闸只在 V2 + SubAgent 时生效，`codex-rs/core/src/agent/control/execution.rs:91-97`：

```rust
fn is_execution_limited(
    multi_agent_version: MultiAgentVersion,
    session_source: &SessionSource,
) -> bool {
    multi_agent_version == MultiAgentVersion::V2
        && matches!(session_source, SessionSource::SubAgent(_))
}
```

区别很本质：**第一个是「你有几个 agent」，第二个是「同时有几个在烧 token」**。一个 agent 可以存在但空闲（已 spawn 但没在跑 turn），占第一道不占第二道。

CAS 抢名额，`codex-rs/core/src/agent/registry.rs:337-353`：

```rust
fn try_increment_spawned(&self, max_threads: usize) -> bool {
    let mut current = self.total_count.load(Ordering::Acquire);
    loop {
        if current >= max_threads {
            return false;
        }
        match self.total_count.compare_exchange_weak(
            current,
            current + 1,
            Ordering::AcqRel,
            Ordering::Acquire,
        ) {
            Ok(_) => return true,
            Err(updated) => current = updated,
        }
    }
}
```

Go 里一模一样：

```go
func (r *AgentRegistry) tryIncrement(max int64) bool {
    for {
        cur := r.totalCount.Load()
        if cur >= max {
            return false
        }
        if r.totalCount.CompareAndSwap(cur, cur+1) {
            return true
        }
    }
}
```

## 5. 通信：三道路径

父子 agent 的任务与结果通过控制面寻址和消息发送；但它们还共享 AgentControl、registry、限额器等并发状态（`agent/control.rs:117-139`），**不能说「没有共享可变状态」**。Go 类比也要给这些共享 map/状态加锁或使用原子变量。

### ① 父 → 子：派活

`codex-rs/core/src/agent/control.rs:194` 的 `send_input` → `thread.start_or_steer_turn(...)`。有两个结果分支（`control.rs:206-216`）：

```rust
match thread
    .start_or_steer_turn(TurnInputRequest::user_input(input).on_start(start_options))
    .await
{
    Ok(TurnInputSubmission::Started { turn_id }) => Ok(turn_id),
    Ok(TurnInputSubmission::Steered { .. }) => {
        // MAv1 exposes an opaque `submission_id` to the model. The legacy
        // `Op::UserInput` path returned a fresh ID for every steer, while the
        // turn-input API returns the active turn ID. Keep the tool-visible ID
        // unique without adding a submission receipt back to Core.
        Ok(Uuid::now_v7().to_string())
    }
    Ok(TurnInputSubmission::NotSubmitted { reason }) => Err(CodexErr::InvalidRequest(
        format!("turn input was not submitted: {reason:?}"),
    )),
    Err(err) => Err(err),
}
```

**同一个入口既是「派新任务」也是「追加指令」**——取决于子 agent 当前是否有活跃 turn。

### ② 子 → 父：终态通知，V1/V2 两条路径

**spawn 后只在非 V2 启动 watcher**（`codex-rs/core/src/agent/control/spawn.rs:800-812`）；它在 `control.rs:626-715` 订阅子状态、等待终态。逐字片段（`:641-656`）：

```rust
            let status = match control.subscribe_status(child_thread_id).await {
                Ok(mut status_rx) => {
                    let mut status = status_rx.borrow().clone();
                    while !is_final(&status) {
                        if status_rx.changed().await.is_err() {
                            status = control.get_status(child_thread_id).await;
                            break;
                        }
                        status = status_rx.borrow().clone();
                    }
                    status
                }
                Err(_) => control.get_status(child_thread_id).await,
            };
            if !is_final(&status) {
                return;
            }
```
```

非 V2 正常路径在 `control.rs:706-714` 向父会话注入 `SubagentNotification`：

```rust
parent_thread
    .inject_fragment_without_turn(SubagentNotification::new(
        child_reference.as_str(),
        status,
    ))
    .await;
```

**正常 V2 路径不是 watcher！** 子 session 发终态回合事件后（`session/mod.rs:2271-2273`），由 `:2292-2338` 检查 `is_final`，在 :2403-2435 直接将 `InterAgentCommunication` 送到父方：

```rust
let communication = InterAgentCommunication::new(
    child_agent_path.clone(),
    parent_agent_path,
    Vec::new(),
    message,
    /*trigger_turn*/ false,
);
```

这里的 `trigger_turn: false` 是「送消息但不要求唤醒空闲父 agent 开新 turn」。`agent/status.rs:26-30` 的 `Interrupted` 不是 final；完成通知不是「所有 turn 退出都发」。`control.rs:663-705` 仍留有 watcher 内部的 V2 防御分支，但不能把它描述成正常 V2 spawn 路径。

Go 示意要分开：

```go
// 非 V2：spawn 后起 watcher，等终态再注入父方上下文。
func watchLegacyCompletion(child *Agent, parent *Agent) {
    go func() {
        final := child.Status.WaitFinal()
        if final.IsFinal() { parent.InjectSubagentNotification(child.ID, final) }
    }()
}

// V2：在子 session 发最终 TurnComplete/TurnAborted 事件的路径中直接转发。
func notifyParentOnTerminalTurn(child *Agent, parent *Agent, event TurnEvent) {
    status := statusFromEvent(event)
    if !status.IsFinal() { return }
    parent.SendAgentMessage(CompletionMessage{
        From: child.Path, To: parent.Path, Status: status, TriggerTurn: false,
    })
}
```

### ③ 查询：wait_agent

`codex-rs/core/src/tools/handlers/multi_agents/wait.rs:307-327`：

```rust
async fn wait_for_final_status(
    session: Arc<Session>,
    thread_id: ThreadId,
    mut status_rx: Receiver<AgentStatus>,
) -> Option<(ThreadId, AgentStatus)> {
    let mut status = status_rx.borrow().clone();
    if is_final(&status) {
        return Some((thread_id, status));
    }
    loop {
        if status_rx.changed().await.is_err() {
            let latest = session.services.agent_control.get_status(thread_id).await;
            return is_final(&latest).then_some((thread_id, latest));
        }
        status = status_rx.borrow().clone();
        if is_final(&status) {
            return Some((thread_id, status));
        }
    }
}
```

默认 30 秒（`tools/handlers/multi_agents_common.rs:31` 的 `DEFAULT_WAIT_TIMEOUT_MS = 30_000`），参数须大于零且被 clamp 到允许范围（`wait.rs:92-100`）：

```rust
let timeout_ms = args.timeout_ms.unwrap_or(DEFAULT_WAIT_TIMEOUT_MS);
let timeout_ms = match timeout_ms {
    ms if ms <= 0 => {
        return Err(FunctionCallError::RespondToModel(
            "timeout_ms must be greater than zero".to_owned(),
        ));
    }
    ms => ms.clamp(MIN_WAIT_TIMEOUT_MS, MAX_WAIT_TIMEOUT_MS),
};
```

Go 版（示意：`WaitFinal` 要正确实现订阅、取消和终态判断；实际 Rust 还会收割同时就绪的其他结果，见 `wait.rs:159-189`）：

```go
func (c *AgentControl) WaitAgent(targets []ThreadID, timeoutMs int) WaitResult {
    // ① 先查有没有已经终态的 —— 快路径，避免漏掉「早就完成了」
    var done []Result
    var watches []*StatusWatch
    for _, id := range targets {
        w, err := c.SubscribeStatus(id)
        if err != nil {
            done = append(done, Result{id, AgentNotFound})  // NotFound 也算终态
            continue
        }
        if status := w.Value(); status.IsFinal() {
            done = append(done, Result{id, status})
        } else {
            watches = append(watches, w)
        }
    }
    if len(done) > 0 {
        return WaitResult{Status: done, TimedOut: false}
    }

    // ② 并发等所有 watch
    ctx, cancel := context.WithTimeout(context.Background(), time.Duration(timeoutMs)*time.Millisecond)
    defer cancel()

    ch := make(chan Result, len(watches))
    for _, w := range watches {
        go func(w *StatusWatch) {
            if r, ok := w.WaitFinal(ctx); ok { ch <- r }
        }(w)
    }

    // ③ 至少等一个终态；实际 Rust 还会收割此刻已就绪的其他终态。
    select {
    case r := <-ch:
        return WaitResult{Status: []Result{r}, TimedOut: false}
    case <-ctx.Done():
        return WaitResult{Status: nil, TimedOut: true}
    }
}
```

## 6. 生命周期：关闭

`codex-rs/core/src/agent/control/legacy.rs` 有三个层次：

| 方法 | 做什么 |
|---|---|
| `shutdown_live_agent` | 发 `Op::Shutdown` → `wait_until_terminated` → 从 registry 摘除 → 释放名额 |
| `close_agent` | 先在持久化层把 spawn 边标记为 `Closed`，再 shutdown 整棵树 |
| `shutdown_agent_tree` | 先收集所有活的子孙，再逐个 shutdown |

`legacy.rs:8-44` 的 `shutdown_live_agent` 顺序值得注意：**先 flush rollout，再发 Shutdown，再等终止，最后才摘除**——保证退出前数据落盘。

## 7. Go 视角下的五个坑

**① panic 行为不同**

Rust/Tokio 的异步 task 在 `panic=unwind` 下可通过 `JoinHandle` 获得 `JoinError`；Go 某个 goroutine 的未恢复 panic 会让**整个进程退出**。如果要做 Go 子任务的 panic 边界，应在启动该 goroutine 的函数中 `defer recover()`，将 panic 转成错误/终态并关闭 done channel；只在另一个 goroutine 的收件循环里 recover **保护不到**独立的回合 goroutine。Rust 若以 `panic=abort` 构建则不能套用 unwind 的结论。

**② 没有 JoinHandle，退出需自己通知**

`codex-rs/core/src/codex_thread.rs:689`：

```rust
pub(crate) fn is_running(&self) -> bool {
    !self.io.tx_sub.is_closed()
}
```

**源码事实：**`codex-rs/core/src/session/mod.rs:1060` 附近把会话循环的 JoinHandle 转为可共享的退出 future：

```rust
pub(crate) fn session_loop_termination_from_handle(
    handle: JoinHandle<()>,
) -> SessionLoopTermination {
    async move {
        let _ = handle.await;
    }
    .boxed()
    .shared()
}
```

此处只把 handle 完成转换为「已终止」信号，**丢弃了 JoinError 与正常退出的区别**；不要推断进程其他地方都完全无法检测错误。

Go goroutine 没有内建的 JoinHandle；若需要等待退出，就在 `SessionIO` 里维护 `done chan struct{}` 并在该 goroutine 退出时 `defer close(done)`（若还需区分错误，再传递退出结果）。

**③ 没有 watch，「最新值 + 广播」要自己封**

tokio 的 `watch` 语义是：只保留最新值，所有订阅者收到变更通知。Go 标准库没有。最地道的替代：

```go
type StatusWatch struct {
    mu  sync.RWMutex
    val AgentStatus
    ch  chan struct{}   // 构造时初始化；变更时 close 旧的，新建一个
}

func (w *StatusWatch) Set(v AgentStatus) {
    w.mu.Lock()
    w.val = v
    close(w.ch)             // 唤醒所有等待者
    w.ch = make(chan struct{})
    w.mu.Unlock()
}

func (w *StatusWatch) Value() AgentStatus {
    w.mu.RLock()
    defer w.mu.RUnlock()
    return w.val
}
```

`close(chan)` 当广播用，是 Go 里常见的技巧。构造时要先初始化 `ch`，等待方必须在同一把锁下同时读取 `val` 与当前 `ch`，否则「读了旧状态再订阅新 channel」会错过终态变更；上面只是状态更新半边。

**④ 没有 Drop，名额回收全靠人记**

Rust 的 `SpawnReservation` 靠 `Drop` 自动还名额。Go 里同样的事必须显式 `defer`，且要注意 `Commit` 之后不能再回滚。忘写的后果是**永久性泄漏**（见第 3 节）。

**⑤ 取消边界不同**

Rust 的 `AbortOnDropHandle` 在 handle drop 时可调用 Tokio 的 task abort；Codex 还使用 `CancellationToken` 做协作式退出（见 `tasks/mod.rs:360-409`）。Go 的 `context.Context` 本身不会强制杀 goroutine；goroutine 不检查 `ctx.Done()` 或没有可取消的阻塞点就不会退出。实现时须明确取消传播、等待与清理顺序。

## 8. 一句话总结

> **Codex 的多 agent = 每个 agent 一个独立 Session/回合任务，父子共用 AgentControl 做配额、寻址与通信；非 V2 watcher 等终态，V2 从子回合终态事件直接通知父。**

而且这个架构有个关键性质——**加一层 agent 不需要改基础的回合采样循环**。模型调用 `spawn_agent` 与调用其他 function tool 都经工具分诊、handler；但 `spawn_agent` 的配置继承、注册、通信与执行容量有自己的专门路径。
