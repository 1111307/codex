# 双层 loop × 多 agent × 工具沙箱：一条 Go 视角的执行链

版本：本 fork `learning` 分支。路径和行号相对 `codex/` 根目录；「观察」表示跨代码片段的推导，不是上游文档的宣称。以下 Go 代码是结构翻译，未覆盖真实服务的所有状态/平台后端，不可直接当安全沙箱使用。

## 总图：两个不同的「spawn」

```text
父 Session 的 submission_loop（收客户端/agent 操作，不等模型采样）
  └─ start_task → RegularTask.run 外层 loop
       └─ run_turn 内层 loop → 模型采样 → tool call
            ├─ spawn_agent → 新的 CodexThread / Session / submission_loop / turn task
            │       └─ 子 agent 自己也有 RegularTask.run → run_turn
            └─ exec_command → 审批 → 选沙箱 → 包装命令 → 启动受限 OS 子进程
```

**不能混淆：**子 agent 是同一 Codex 进程里的新 session/task（类比 Go 的 goroutine）；命令沙箱是执行命令时创建/配置的 OS 子进程。子 agent 的创建并不等于立刻 fork 一个隔离容器，也不等于启动沙箱进程。

## 1. 调度入口：submission_loop 不堵住审批/插话

`codex-rs/core/src/session/mod.rs:574-575`：

```rust
let (tx_sub, rx_sub) = async_channel::bounded(SUBMISSION_CHANNEL_CAPACITY);
let (tx_event, rx_event) = async_channel::unbounded();
```

`mod.rs:877-883` 在 Tokio task 中跑 `submission_loop`，`session/handlers.rs:411-418` 从 `rx_sub.recv().await` 循环取操作（直到 Shutdown 或 channel 关闭）；`:471-478` 处理 `Op::TurnInput`，`:529-542` 同一个循环处理审批回复、用户提问答复。`session/turn_input.rs:269-333` 判断输入是 steer（仍在跑的 turn）还是 started，后者在 :328 调用 `spawn_task(... RegularTask::new())`。`tasks/mod.rs:360-409` 再把真正的回合跑在**另一个** tokio task。**因此：模型/工具/审批挂起时，submission_loop 还能收到后续操作。**

Go 结构：

```go
// 示意：实际 Op、生命周期、错误处理、同步保护均需实现。
func (s *Session) submissionLoop(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        case op, ok := <-s.submissions:
            if !ok { return }
            switch op.Kind {
            case StartTurn:
                // 别在这儿直接跑模型：会堵住 Approval/Steer。
                s.startSingleActiveTurn(ctx, op.Input) // 内部新起 goroutine
            case Steer:
                s.pending.Append(op.Input)
            case ApprovalReply:
                s.notifyApproval(op.ApprovalID, op.Decision)
            case Shutdown:
                return
            }
        }
    }
}
```

Go 的 `chan` 不能直接替代 Rust 无界事件通道；若照搬 `make(chan Event)`，慢客户端会阻塞事件发送。需要选定缓冲、独立转发器、持久化或背压策略，不能笼统称 `chan Event` 为「无界」。

## 2. 双层回合：外层再进 turn，内层再采样

**外层** `tasks/regular.rs:73-96`：

```rust
let mut next_input = input;
let mut prewarmed_client_session = prewarmed_client_session;
let mut mcp_startup_requirements = McpStartupRequirements::default();
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

注意末尾的空输入是「让下一次 `run_turn` 自己从 queue 取」的标记，而不是「去空 channel 阻塞等新输入」。

**内层** `session/turn.rs:365, 414-435, 510-565, 600-704`：第一次先采样本轮显式输入，后续按 `can_drain_pending_input` 拉插话；每步从历史拼 prompt（:510-516），再调用 `run_sampling_request`（:522-532）；流事件处理把工具 future 放进 `in_flight`（:2627-2641），执行后以 `model_needs_follow_up || has_pending_input` 决定是否再迭代（:565）；若须换上下文窗口，中途压缩后继续（:600-639）；都不需要续跑则处理 stop hooks，最后 break（:642-704）。`session/input_queue.rs:291-323` 的 `get_pending_input` 从 `pending_input.items.split_off(0)`/mailbox **取已有数据**，不会等待用户将来输入。

Go 骨架（省略具体 hook、压缩判定和记录协议）：

```go
func (s *Session) regularTask(ctx context.Context, initial []Input) (string, error) {
    next := initial
    for {
        last, err := s.runTurn(ctx, next)
        if err != nil { return "", err }
        if s.terminalError() != nil || !s.pending.Has() { return last, nil }
        next = nil // 对应 Vec::new：下一轮由 runTurn 拉 pending
    }
}

func (s *Session) runTurn(ctx context.Context, input []Input) (string, error) {
    drain := len(input) == 0
    s.recordInputs(input)
    for {
        if err := ctx.Err(); err != nil { return "", err }
        if drain { s.recordInputs(s.pending.Take()) } // 取已有项，不是 <-ch 等输入
        step, err := s.sampleAndExecuteTools(ctx, s.historyForPrompt())
        if err != nil { return "", err }
        drain = true
        needsFollowup := step.HasToolContinuation || s.pending.Has()
        if needsFollowup && s.shouldCompact() {
            if err := s.compact(ctx); err != nil { return "", err }
            drain = !step.HasToolContinuation // 对应 turn.rs:638
            continue
        }
        if needsFollowup { continue }
        outcome := s.runStopHooks()
        if outcome.Blocked {
            if prompt, ok := buildContinuationPrompt(outcome.Fragments); ok {
                s.record(prompt)
                continue // 对应 turn.rs:661-678；只有有续跑 prompt 才继续
            }
            s.warn("Stop hook requested continuation without a prompt; ignoring the block.")
        }
        if outcome.ShouldStop { return step.LastAgentMessage, nil } // 对应 turn.rs:689-702；简化：省略 legacy after-agent hook 等分支
        return step.LastAgentMessage, nil
    }
}
```

「观察」：这段翻译刻意把 pending 设成加锁 slice/队列，而不是空时阻塞的 channel——**插话是 PULL**；只有等待审批/工具 `sleep` 等才有阻塞等待的场景。上面仅画出两层业务 loop，此外还有最外层的 submission_loop，以及 `run_sampling_request` 自身的网络重试循环（`turn.rs:1561-1607`），不要以为代码里只有两个 `loop`。

## 3. 多 agent：同一种 Session 套娃，不是子 OS 进程

调用链：`tools/router.rs:244-288` 将模型 function call 分派为 tool call → `tools/handlers/multi_agents/spawn.rs:47-133` 解析参数、检验 depth（:68-74）、从当前 turn 构造子配置（:92-109）并调用 `AgentControl::spawn_agent_with_metadata`（:111-133）。超深度按 `FunctionCallError::RespondToModel` 返给模型，不是杀进程。

`tools/handlers/multi_agents_common.rs:170-176` 的注释原话（描述 spawn config builder 的共通用途；但完整 fork 会拒绝 agent type 覆盖，且不应用 role overlay，见 `multi_agents/spawn.rs:94-107`）：

```rust
/// The returned config starts from the parent's effective config and then refreshes the
/// runtime-owned fields carried by the turn, including model selection, reasoning settings,
/// approval policy, sandbox, and cwd. Role-specific overrides are layered
/// after this step; skipping this helper and cloning stale config state directly can send the child
/// agent out with the wrong provider or runtime policy.
```

`:195-220` 刷新模型、provider、reasoning、developer instructions；`:238-264` 复制 live turn 的审批策略、cwd、permission profile snapshot。非 fork 路径才由 `spawn.rs:105-107` 应用用户 role（可覆盖字段见 `agent/role.rs:79-89`）；完整 history fork 会拒绝 `agent_type` 覆盖（`:94-96`），不能笼统说 role 总会叠加。**子 agent 继承的是当前生效权限配置，不是重新从本地默认值计算一遍。** 若 `fork_context=true`，`control/spawn.rs:684-695` 走 fork 历史路径，fork 前会 flush 父历史并读取上下文（`:883-912`）；否则 :697-722 建新会话。默认不要误认为完整复制父历史。

`agent/control.rs:117-139`：一棵 root agent 树共用一个 `AgentControl`（registry、限额等）；`agent/control/spawn.rs:631-652` 检查执行容量、可选预留 V2 residency slot，再预留 spawn 配额；`:684-725` 用**同一个控制面**调用 ThreadManager 建新 thread；`thread_manager.rs:2096-2155` 调用 `Session::spawn`；`session/mod.rs:877-883` 起子 session 的 submission_loop；`thread_manager.rs:2180-2213` 收到 `SessionConfigured` 才把它放进全局 thread map；`agent/control/spawn.rs:726-799` 注册子 agent 后送首条输入。子 session 的 turn 同样由 `RegularTask` 驱动。

`agent/registry.rs:95-115, 385-401` 的 `SpawnReservation`：**commit 之前**失败 `Drop` 归还名额；成功 `commit` 将名额登记给活跃 agent。commit 后若初始输入派发失败，不能靠 reservation 的 Drop 回收；上面的 Go 示意显式清理，这并非对当前 Rust 失败清理路径的验证。`agent/control/execution.rs:62-97` 另有 V2 子 agent 的并行 turn 执行限额，和「活着的 agent 个数」是两条不同的闸。

Go 结构翻译：

```go
type Control struct {
    registry *Registry   // 每棵 agent 树共享：路径、活 agent 数
    limiter  *Limiter    // V2 子 agent：同时运行 turn 的数目
}
type Agent struct {
    ID ThreadID
    Session *Session
    Control *Control
    Done chan struct{}   // Go 无 JoinHandle，退出时显式 close
}

// 示意，不是可直接运行的实现：
// parent 当前配置视为已从 turn 的有效快照刷新；完整 fork 不应用 role 覆盖。
func (c *Control) Spawn(parent *Agent, req SpawnRequest) (*Agent, error) {
    if parent.Depth+1 > parent.MaxDepth { return nil, ErrDepthLimit }
    cfg := cloneEffectiveTurnConfig(parent.Session) // 模型、审批、cwd、权限快照
    if req.ForkContext {
        if req.Role != "" { return nil, ErrFullForkAgentTypeOverride }
        // 完整历史 fork 继承父 agent type，不叠加 role
    } else {
        cfg.ApplyRole(req.Role) // 仅允许的 role 覆盖项
    }
    slot, err := c.registry.Reserve(cfg.MaxAgents)
    if err != nil { return nil, err }
    committed := false
    defer func() { if !committed { slot.Rollback() } }()

    child, err := startThreadAndWaitConfigured(cfg, c) // 内部启动 submissionLoop
    if err != nil { return nil, err } // 不执行 exec.Command，也不启动 bwrap
    slot.Commit(child.ID)
    committed = true
    if err := child.StartOrSteer(req.Input); err != nil {
        child.ShutdownAndRelease() // Go 设计：commit 后失败须显式回收
        return nil, err
    }
    return &Agent{ID: child.ID, Session: child, Control: c, Done: child.Done}, nil
}
```

子到父的结果：非 V2 `agent/control/spawn.rs:800-811` 起独立 watcher，`agent/control.rs:626-715` 订阅子状态，终态后注入父的上下文；**正常 V2 路径不靠这个 watcher**，而是子 session 在发 `TurnComplete`/`TurnAborted` 时直接通知父（`session/mod.rs:2292-2338, 2403-2435`），消息 `trigger_turn: false`——送上下文，但不自动让空闲的父开新 turn。`agent/status.rs:26-30` 中 Interrupted 不是 final，可被后续输入继续使用。

## 4. 沙箱：agent 建立时带策略，执行命令时才选后端

旧式三档 `SandboxMode::{ReadOnly, WorkspaceWrite, DangerFullAccess}` 在 `protocol/src/config_types.rs:104-114`；`config/src/config_toml.rs:751-828` 在排除 `default_permissions` 后把旧式配置映射为 `PermissionProfile`。**另有命名 `[permissions]` 配置路径，不必经过 `sandbox_mode`**（`config_toml.rs:751-755`）。在普通预设中，ReadOnly 从 `protocol/src/permissions.rs:594-611` 得到 `Root/Read`；WorkspaceWrite 从 :799-853 得到 `Root/Read` 加 project roots、临时目录等可写，`.git/.agents/.codex` 默认保护。**不要推广为「所有配置都全盘可读」**：自定义权限可有 `Deny` 条目（`permissions.rs:676-687`）；Linux bwrap 遇非全盘读策略会改为 `--tmpfs /` + 逐项 `--ro-bind`（`linux-sandbox/src/bwrap.rs:382-395, 471-539`）。

对一次本地普通 `exec_command`，实读调用链：

1. `tools/handlers/unified_exec/exec_command.rs:154-205` 解析参数/选择 environment、准备 cwd；`unified_exec/process_manager.rs:1389-1390` 构造 `ToolOrchestrator` + runtime。
2. `tools/orchestrator.rs:141-235` 决定是否要审批；`:237-307` 依据当前 `TurnEnvironment` 的权限与网络策略挑选第一次执行沙箱（`SandboxManager::should_sandbox/select_initial` 在 `sandboxing/src/manager.rs:317-356`）。审批策略和沙箱策略是两回事：**审批通过不等于绕过沙箱**。
3. `core/src/tools/sandboxing.rs:444-475` 的 `SandboxAttempt::env_for` 调 `SandboxManager::transform`；`sandboxing/src/manager.rs:392-465` 把原始命令变换成 macOS `sandbox-exec` / Linux `codex-linux-sandbox` 包装命令；Windows 走受限 token 子进程的后端（:466-490）。Linux 的 bwrap mount 规则是按权限生成的，不是 `chdir` 或 Go `filepath` 检查。
4. `unified_exec/process_manager.rs:1192-1235, 1248-1359` 将准备好的 request 交本地 `codex_sandboxing::spawn_process`；远程环境/特殊 shell snapshot 走 exec-server，执行端在自己的环境施加策略（:1209-1213, 1261-1286）。本地 spawn 定义在 `sandboxing/src/spawn.rs:44-66`。`core/src/exec.rs:882-944` 也明示底层 `exec` **不负责沙箱化**，调用方必须先构造包装命令。
5. `tools/orchestrator.rs:322-490`：若第一次是 SandboxDenied，**不是无条件重跑裸命令**。只有条件符合、允许提权且获得必要审批时才有第二次 attempt；`Never/OnRequest` 的常规拒绝路径会直接返回原 sandbox denial（:367-400）。

Go 等价分层：

```go
// 只画控制面；buildLaunch 必须调用 OS 级 bwrap/Seatbelt/受限令牌后端，
// 不能用 filepath 检查或 os.Chdir 冒充安全边界。
func runCommand(ctx context.Context, env TurnEnvironment, request Command) (Output, error) {
    profile := env.PermissionProfile.MaterializeRoots(env.WorkspaceRoots)
    if err := approveIfRequired(ctx, profile, request); err != nil { return Output{}, err }
    kind := selectSandbox(profile, env.Platform, request)
    launch, err := buildLaunch(profile, kind, request)
    if err != nil { return Output{}, err }
    out, err := spawnOSProcess(ctx, launch) // Linux 包装程序 / macOS sandbox-exec / Windows 受限 token
    if !isSandboxDenied(err) { return out, err }
    if !canEscalate(profile, request) { return out, err }
    if err := approveEscalationIfRequired(ctx, request); err != nil { return out, err }
    retryLaunch, err := buildEscalatedLaunch(profile, request)
    if err != nil { return out, err }
    return spawnOSProcess(ctx, retryLaunch)
}
```

**最值得记的一句：**`spawn_agent` 启动的是一个会思考和接任务的 session；`exec_command` 才启动 OS 子进程，并且**每次工具执行尝试**以当时生效的 permission profile 判断是否需要沙箱、选择可用后端（也可能是 `None`）。前者继承权限状态，不直接调用沙箱后端。

## 没验证的

- Go 代码是示意模型，没有编译或连接真实模型，也没有实现强制沙箱。不要拷贝到生产环境作为权限边界。
- 这里只逐段核对了普通的 spawn_agent / 普通本地 exec_command 主路径及远程分支入口，没穷尽所有工具、exec-server 实现、Windows 受限 token 的实现细节和 V2 residency 逐出恢复的全部条件。
- 完成通知 V1/V2、legacy 读权限的描述对应这份 fork 的实现；未来版本或特殊后端应重新核对，不能直接推广。
