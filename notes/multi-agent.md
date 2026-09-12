# 多智能体架构

一句话：**不是「多线程」，是「多 goroutine + 消息传递」，每个 agent 就是一个完整的独立 session。** 父和子没有本质区别，只是 `AgentPath` 和 `depth` 不同。

## 0. Rust/Tokio → Go 对照表

| Codex (Rust/Tokio) | Go 里的对应物 | 差异要点 |
|---|---|---|
| `tokio::spawn(fut)` | `go f()` | 基本等价 |
| `async_channel::bounded(n)` | `make(chan T, n)` | 一样，有背压 |
| `Arc<Session>` | `*Session` | Go 有 GC，不用手动 Arc |
| `Mutex<HashMap>` | `sync.Mutex` + `map` | 一样 |
| `AtomicUsize` + `compare_exchange_weak` | `atomic.Int64.CompareAndSwap` | 一样 |
| `watch::Receiver<T>` | **没有直接对应** | 要自己封「最新值 + 广播」 |
| `CancellationToken` | `context.Context` | 都是协作式取消 |
| `Drop` / `AbortOnDropHandle` | `defer` | Go 的 defer **不能**中止 goroutine |
| `Weak<ThreadManagerState>` | **没有直接对应** | Go GC 能处理环，弱引用要 `SetFinalizer` |
| `JoinHandle` | **没有** | goroutine 退出收不到通知 |

最后三行 Go 缺的东西，恰好是这套架构依赖最深的机制。见第 7 节。

## 1. 三层结构

```
ThreadManagerState        全局注册表，一进程一份
  map[ThreadID]*Agent + rollout store + graph store
        ↑ Weak 引用（打破循环）
AgentControl              一棵 agent 树共享一份
  registry: map[path]*AgentMeta, totalCount, limiter
        ↑ Clone 给每个 agent
Agent (goroutine)  ×N     每个 = 独立 Session + 独立 rollout + 独立信箱
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

**一个 root session 一个 AgentControl** —— 配额是「每个用户会话」的，不是全局的。开两个 Codex 窗口，各自独立计数。

`manager: Weak<ThreadManagerState>` 是弱引用，注释里写明原因：

```rust
/// This is `Weak` to avoid reference cycles and shadow persistence of the form
/// `ThreadManagerState -> CodexThread -> Session -> SessionServices -> ThreadManagerState`.
```

## 2. 一个 agent = 一个 goroutine + 两个 channel

SQ/EQ 协议（Submission Queue / Event Queue）的物理实现，`codex-rs/core/src/session/mod.rs:574-575`：

```rust
let (tx_sub, rx_sub) = async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY);
let (tx_event, rx_event) = async_channel::unbounded();
```

**一有一无是刻意的**：入站有界，防止调用方灌爆；出站无界，防止 agent 卡在发事件上。

Go 版：

```go
type SessionIO struct {
    sub    chan Op       // 入口：外面往里投指令  (bounded, 有背压)
    events chan Event    // 出口：agent 往外吐事件  (unbounded, 不阻塞)
    done   chan struct{} // agent 退出时 close —— 补 Go 缺失的 JoinHandle
}
```

主循环，对应 `codex-rs/core/src/session/mod.rs:877-890` 的 `tokio::spawn(submission_loop(...))`：

```rust
// This task will run until Op::Shutdown is received.
let session_for_loop = Arc::clone(&session);
let session_loop_handle = tokio::spawn(async move {
    submission_loop(session_for_loop, configured_config, rx_sub)
        .instrument(info_span!("session_loop", thread_id = %thread_id))
        .await;
});
```

Go 版：

```go
func (a *Agent) Run() {
    defer close(a.IO.done)
    defer a.status.Set(AgentShutdown)

    // 把 panic 关在当前 agent 里 —— Go 里这行是必须的！
    defer func() {
        if r := recover(); r != nil {
            a.status.Set(AgentErrored(fmt.Sprint(r)))
        }
    }()

    for {
        select {
        case op, ok := <-a.IO.sub:
            if !ok {
                return
            }
            switch op.Kind {
            case OpUserInput:
                a.RunTurn(op)               // 模型请求 → 工具 → 再请求的循环
            case OpInterAgentCommunication:
                a.HandlePeerMessage(op)
            case OpInterrupt:
                a.CancelTurn()              // 等价于 cancel ctx
            case OpShutdown:
                return                      // ← 循环唯一的正常出口
            }
        }
    }
}
```

**父 agent 和子 agent 跑的是同一个函数。**

## 3. spawn_agent 全流程

模型调 `spawn_agent` 时，它就是个普通 tool handler：`codex-rs/core/src/tools/handlers/multi_agents/spawn.rs:47` 的 `handle_spawn_agent`，背后调 `AgentControl::spawn_agent_internal`（`codex-rs/core/src/agent/control/spawn.rs:614`）。

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

Go 版全流程：

```go
func (c *AgentControl) SpawnAgent(cfg Config, input []UserInput, src SessionSource) (*LiveAgent, error) {
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

    // ③ 起一个新的 agent goroutine —— spawn.rs:709
    agent := c.manager.NewThread(cfg, c, src)
    go agent.Run()           // ← 就是那个 tokio::spawn(submission_loop(...))

    // ④ 登记进 registry
    meta := AgentMeta{ID: agent.ID, Path: agent.Path}
    slot.Commit(meta)
    committed = true

    // ⑤ 投第一句话，agent 开始动 —— spawn.rs:784-788
    agent.IO.sub <- Op{Kind: OpUserInput, Items: input}

    return &LiveAgent{ThreadID: agent.ID, Status: agent.Status()}, nil
}
```

**`slot + committed` 这个模式是 Go 里必须显式写的**。Rust 的 `SpawnReservation` 在 `Drop` 里检查 `self.active`（`registry.rs:393-402`）：

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

意义是**异常安全的名字占用**：抢到名额 → 中途任何一步失败 → 名额自动归还。Go 里忘写那个 `defer`，一次 spawn 失败会永久漏掉一个名额，跑到 `max_threads` 之后整个会话再也开不出 agent，而且没有任何报错。

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
            current, current + 1, Ordering::AcqRel, Ordering::Acquire,
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

Go 的哲学 *"Do not communicate by sharing memory; share memory by communicating."* —— Codex 完全就是这个路子。**agent 之间没有共享可变状态，全靠往对方信箱投消息。**

### ① 父 → 子：派活

`codex-rs/core/src/agent/control.rs:194` 的 `send_input` → `thread.start_or_steer_turn(...)`。有两个结果分支（`control.rs:206-216`）：

```rust
match thread.start_or_steer_turn(TurnInputRequest::user_input(input).on_start(start_options)).await {
    Ok(TurnInputSubmission::Started { turn_id }) => Ok(turn_id),
    Ok(TurnInputSubmission::Steered { .. }) => Ok(Uuid::now_v7().to_string()),
    Ok(TurnInputSubmission::NotSubmitted { reason }) => Err(CodexErr::InvalidRequest(
        format!("turn input was not submitted: {reason:?}"),
    )),
    Err(err) => Err(err),
}
```

**同一个入口既是「派新任务」也是「追加指令」**——取决于子 agent 当前 busy 不 busy。

### ② 子 → 父：干完了主动汇报

独立的 watcher goroutine，`codex-rs/core/src/agent/control.rs:626-716`：

```rust
fn maybe_start_completion_watcher(
    &self,
    child_thread_id: ThreadId,
    session_source: Option<SessionSource>,
    child_reference: String,
    child_agent_path: Option<AgentPath>,
) {
    // ...
    let control = self.clone();
    tokio::spawn(async move {
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
        // ...
    });
}
```

V2 路径（`control.rs:687-703`）：

```rust
let communication = InterAgentCommunication::new(
    child_agent_path,
    parent_agent_path,
    Vec::new(),
    message,
    /*trigger_turn*/ false,
);
let context = AgentCommunicationContext::new(AgentCommunicationKind::Result, child_thread_id);
let _ = control
    .send_inter_agent_communication(parent_thread_id, communication, context, TurnStartOptions::default())
    .await;
```

V1 路径（`control.rs:709-714`）：

```rust
parent_thread
    .inject_fragment_without_turn(SubagentNotification::new(
        child_reference.as_str(),
        status,
    ))
    .await;
```

**V1/V2 的区别**：V1 是「往父的上下文插一段通知」（单向、上下文污染），V2 是「给父发一条正经的 agent 间消息」（有 from/to、有 `trigger_turn` 语义）。这是个明显的架构演进。

`trigger_turn: false` 很重要——**只注入上下文，不唤醒父 agent 开新 turn**。

Go 版：

```go
func (c *AgentControl) startCompletionWatcher(child *Agent, parentID ThreadID) {
    go func() {   // ← 独立 goroutine，不是父 agent 轮询
        final := child.status.WaitFinal(context.Background())
        if !final.IsFinal() {
            return
        }
        c.SendInterAgentCommunication(parentID, InterAgentCommunication{
            From:        child.Path,
            To:          parentPath,
            Message:     formatCompletionMessage(child.Path, parentPath, final),
            TriggerTurn: false,
        })
    }()
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

超时默认 30 秒，被 clamp（`wait.rs:92-100`）：

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

Go 版（标准并发模式）：

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
        if w.Value().IsFinal() {
            done = append(done, Result{id, w.Value()})
        } else {
            watches = append(watches, w)
        }
    }
    if len(done) > 0 {
        return WaitResult{Status: done, TimedOut: false}
    }

    // ② 并发等所有 watch
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    ch := make(chan Result, len(watches))
    for _, w := range watches {
        go func(w *StatusWatch) {
            ch <- w.WaitFinal(ctx)
        }(w)
    }

    // ③ 拿到第一个就返回，不是等所有
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

**① panic 行为完全相反**

```rust
// Rust: task panic → 该 task 死，进程活  → 靠 JoinError 感知
// Go:   goroutine panic → 整个进程死      → 必须 recover
```

Rust 里 `tokio::spawn` 的 task panic 被 catch 在 task 边界内（默认 `panic=unwind`，Codex 也没开 `panic="abort"`）。**Go 里不 recover 就是全进程崩。** 所以 Go 版必须在 `Run()` 里加 `defer recover()`，否则一个子 agent 出 bug 会拖垮所有 agent——而这个隔离性正是多 agent 架构最重要的性质之一。

**② 没有 JoinHandle，退出不可见**

`codex-rs/core/src/codex_thread.rs:689`：

```rust
pub(crate) fn is_running(&self) -> bool {
    !self.io.tx_sub.is_closed()
}
```

**这是被逼出来的设计**——因为 `codex-rs/core/src/session/mod.rs:1060` 那里，JoinHandle 的返回值直接被扔了：

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

`let _ = handle.await;` 意味着 **panic 和正常退出无法区分**。

Go 里更彻底——goroutine 退出了你完全不知道。所以必须在 `SessionIO` 里自己维护 `done chan struct{}` 并 `defer close(done)`。

**③ 没有 watch，「最新值 + 广播」要自己封**

tokio 的 `watch` 语义是：只保留最新值，所有订阅者收到变更通知。Go 标准库没有。最地道的替代：

```go
type StatusWatch struct {
    mu  sync.RWMutex
    val AgentStatus
    ch  chan struct{}   // 变更时 close 旧的，新建一个 —— close 是广播
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

`close(chan)` 当广播用，是 Go 里非常经典的技巧。

**④ 没有 Drop，名额回收全靠人记**

Rust 的 `SpawnReservation` 靠 `Drop` 自动还名额。Go 里同样的事必须显式 `defer`，且要注意 `Commit` 之后不能再回滚。忘写的后果是**永久性泄漏**（见第 3 节）。

**⑤ 取消是协作式的**

Rust 的 `AbortOnDropHandle`（`codex-rs/core/src/tools/parallel.rs:150`）是 drop 即取消，你不可能忘。Go 的 `context.Context` 是协作式的——goroutine 不检查 `ctx.Done()` 就永远不退出。

两者其实都在 await 点生效，但差别在**「能不能忘」**：Rust 里是类型系统保证的，Go 里是纪律保证的。

## 8. 一句话总结

> **Codex 的多 agent = 每个 agent 一个 goroutine + 两个 channel，用消息传递代替共享内存，用一个共享的 AgentControl 做配额和寻址，用独立的 watcher goroutine 做完成通知。**

而且这个架构有个很关键的性质——**加一层 agent 不需要改主循环**。父 agent 调 `spawn_agent` 和调 `read_file` 走的是同一条路：`build_tool_call` → tool handler → 返回结果。多 agent 没有任何特殊路径。
