# exec_command 调用链：审批挂起 → 沙箱选择 → 子进程 spawn

围绕一个问题：模型发出一条本地 `exec_command`、这条命令需要用户审批、用户点了批准——从工具调用到 OS 子进程跑起来，中间发生了什么？结论先行：**审批发生在 spawn 之前（拒绝不会留下进程）；普通审批通过不等于绕过沙箱（首次沙箱由权限策略另行选择）；被沙箱包住的是工具启动的 OS 子进程，不是 Codex 控制进程本身。** 回合循环与审批挂起的机制细节见 [turn-loop.md](turn-loop.md)，权限预设的静态语义见 [sandbox-permissions.md](sandbox-permissions.md)；本文只追这条调用链。

## 1. 主路径时序

```
模型输出 tool call → 分诊入队 → 并行派发（可取消）
  → ExecCommandHandler 预处理 → process_manager 组请求
  → ToolOrchestrator：①审批（挂起等用户）②选首次沙箱 ③首次执行
  → runtime 包装 SandboxCommand → SandboxManager::transform 平台变换
  → spawn_process（PTY / pipe / Windows 受限 token）
  → from_spawned 检测沙箱拒绝 → 收集输出 → 返回模型
```

### 阶段 0：工具调用怎么进入执行器

回合循环在 `ResponseEvent::OutputItemDone` 时调用分诊函数，产出的工具 future 进 `in_flight`（`session/turn.rs:2627-2637`）。分诊点把 `ResponseItem::FunctionCall` 解析成 `ToolCall`，派生子 cancellation token 后交给工具运行时（`stream_events_utils.rs:300-341`，token 在 `:336`，future 在 `:337-341`）：

```rust
let cancellation_token = ctx.cancellation_token.child_token();
let tool_future: InFlightFuture<'static> = Box::pin(
    ctx.tool_runtime
        .clone()
        .handle_tool_call(call, cancellation_token),
);
```

并行执行器把派发包成 `tokio::spawn` 的任务（`tools/parallel.rs:150-198`），等待 ready、按 `supports_parallel` 取读/写锁后进入 router 分派。**取消 select 在这里**（`parallel.rs:202-234`）——dispatch 完成 vs `cancellation_token.cancelled()`；取消分支会 abort 派发任务并构造 aborted 工具响应（`:210-232`）。注意这是 tokio 任务级 abort，不是 OS 子进程终止。

### 阶段 1：handler 预处理（`exec_command.rs:154-483`）

handler 解析参数、cwd、shell、turn environment，组装 `ExecCommandRequest`（`:421-440`）。两个容易看错的地方：

- `:209-217` 有一处 `SandboxManager::new().select_initial(...)`，注释明说**只是 cwd 一致性预检**（判断是否需要 host 本地 native cwd），不是最终的沙箱选择：

  ```rust
  // Remote executors enforce URI-native sandbox policy themselves. Only a host-local
  // sandbox needs a native cwd for resolving paths nested in the permissions config.
  let requires_host_native_cwd = !environment.is_remote()
      && SandboxManager::new().select_initial(...)
  ```

- 执行模式分两支（`:441-447`）：one-shot（带 `timeout_ms` 的完成语义）走 `exec_command_to_completion`；交互式（默认 TUI）走 `manager.exec_command`。one-shot 内部另有一个 `biased` 的取消/超时 select，取消时会终止已 spawn 的进程（`unified_exec/oneshot.rs:26-86`，select 在 `:57-74`）。

结果映射在 `:448-482`：正常 `Ok` 包装为工具输出；`SandboxDenied` **不是错误**，转成普通工具输出让模型看到拒绝文本（`:450-474`，注释原话「Sandbox denial is terminal, so there is no live process for write_stdin to resume」）；其他错误才走 `RespondToModel`（`:475-481`）。

### 阶段 2：审批判定发生在 spawn 之前

交互式路径 `process_manager.rs:488-506` 的 `exec_command → exec_command_inner → open_session_with_sandbox`。`open_session_with_sandbox`（`:1362-1476`）先组装 env 与请求，再用 exec policy 算出**三态审批需求**（`:1404-1427`）：

```rust
let exec_approval_requirement = context
    .session
    .services
    .exec_policy
    .create_exec_approval_requirement_for_shell(...)
    .await;
```

`ExecApprovalRequirement` 是 `Skip { bypass_sandbox }` / `NeedsApproval { reason }` / `Forbidden { reason }`。之后才进 orchestrator（`:1456-1459`）——**此时还没 spawn 任何进程**。

### 阶段 3：ToolOrchestrator——审批挂起点

文件头注释（`orchestrator.rs:1-8`）就写明编排顺序「approval → select sandbox → attempt → retry with an escalated sandbox strategy on denial (no re-approval thanks to caching)」。`tools/orchestrator.rs:141-235` 按三态分派：

- `Skip`：`strict_auto_review` 开着仍走完整审批链（`:179-200`）；否则记 otel 直接放行（`:202-208`）。
- `Forbidden`：直接 `ToolError::Rejected`（`:210-212`）。
- `NeedsApproval`（本文主路径）：构造 `ApprovalAction::ExecCommand` 后调 `request_approval`（`:213-234`），成功则置 `already_approved = true`。

审批链是三级优先级（`tools/approvals.rs:493-495` 注释原话，hooks → Guardian（若开）→ 用户）。用户路径带缓存：`with_cached_approval`（`tools/sandboxing.rs:70-116`）命中 `ApprovedForSession` 就跳过弹窗。真正等用户回复时，`request_command_approval` 先登记 oneshot sender、再发 `ExecApprovalRequest` 事件、最后挂起在 `rx_approve.await`（`session/mod.rs:2789-2876`）。**挂起与唤醒的机制（Abort 走 interrupt、waiter 表清理、Abort 顺序）在 [turn-loop.md](turn-loop.md) 已详述，此处不重复。**

用户答复经 `Op::ExecApproval` 在 submission loop 分派（`session/handlers.rs:529-536`）；`exec_approval` 里 `Abort` 走 `interrupt_task()`，其他决议走 `notify_approval` 唤醒 waiter（`handlers.rs:196-201`）。决议到工具错误的映射在 `approvals.rs:434-463` 的 `into_tool_result`：`Denied` → `Rejected`，`TimedOut` → `Rejected`，`Abort` → `CodexErr::TurnAborted`，`Approved*` → 继续。

### 阶段 4：批准之后，才选择首次沙箱

审批通过**不改变**首次沙箱选择（`orchestrator.rs:237-312`）。`already_approved` 的唯一用途是后面升级重试时免二次审批（`:414-416`，见 `tools/sandboxing.rs:315-321` 的 `should_bypass_approval`）。首次选择由这组逻辑决定：

- `unsandboxed_execution_allowed`（`tools/sandboxing.rs:275-279`）：策略含 denied-read 时禁止脱沙箱——脱了沙箱 deny-read 就没人执行了。
- `sandbox_override_for_first_attempt`（`:238-267`）：只有 exec policy `Skip { bypass_sandbox: true }`（显式全信任规则）或请求了提权时才 `BypassSandboxFirstAttempt`。
- 其余情况 `should_sandbox` + `select_initial`（`orchestrator.rs:270-287`）按权限策略选后端：macOS Seatbelt / Linux seccomp / Windows 受限 token，或 `None`。

所以：**普通审批回答「允不允许执行」，沙箱策略回答「以什么限制执行」**。批准后的命令通常仍在沙箱里跑。

### 阶段 5：构造命令并 spawn OS 子进程

runtime 把 shell 向量包装成 `SandboxCommand`（`tools/runtimes/unified_exec.rs:131-149` 的 `build_unified_exec_sandbox_command`，调用处 `:681-693`），然后交回 process manager（`:695-713`）。本地路径在 `process_manager.rs:1192-1234` 的 `open_session_with_exec_env` 里经 `attempt.env_for` 请求平台变换，落到 `open_session_with_prepared_exec_env`（`:1248-1360`）：远程环境走 exec-server（`:1261-1287`）；本地在 `:1344-1355` 调 `codex_sandboxing::spawn_process`。

平台变换的实现在 `sandboxing/src/manager.rs:358-521` 的 `SandboxManager::transform`：

- `SandboxType::None`：原样透传（`:392-393`）。
- macOS：命令变成 `/usr/bin/sandbox-exec` + profile 参数 + 原命令（`:394-432`）。
- Linux：命令变成 `codex-linux-sandbox` helper + 生成的参数，并设 arg0 override（`:435-465`）。
- Windows：argv 不变，约束在 spawn 阶段施加（`:466-497`）。

真正的 OS 子进程诞生在 `sandboxing/src/spawn.rs:45-143`：Windows 受限 token 走专用后端（`:55-104`）；其余平台按 tty / stdin_open 选 PTY 或 pipe spawn（`:106-142`）。

### 阶段 6：spawn 后的检测与输出

`UnifiedExecProcess::from_spawned`（`unified_exec/process.rs:343-398`）启动输出转发任务，若早期退出立即查沙箱拒绝；`check_for_sandbox_denial`（`:292-341`）用 executor 上报 + exit code/输出文本启发式（`is_likely_sandbox_denied`）判定。命中则产生 `UnifiedExecError::sandbox_denied`，一路上映射见阶段 1 的结果映射。

## 2. 三条分支

- **用户 Denied**：`into_tool_result` 映射为 `ToolError::Rejected`（`approvals.rs:456`）→ orchestrator 直接返回 → `open_session_with_sandbox` 的错误映射（`process_manager.rs:1460-1475`）→ handler `:475-481` 变成给模型的错误消息。审批在 spawn 前，**拒绝路径不产生进程**。
- **用户 Abort**：`handlers.rs:197-199` → `interrupt_task` → turn 级取消与 waiter 清理，完整机制见 [turn-loop.md](turn-loop.md)。
- **SandboxDenied 升级重试**（`orchestrator.rs:314-517`）：首次沙箱尝试失败且错误为 `Sandbox(Denied)` 时，**不是无条件裸跑重试**——依次过门：`escalate_on_failure()`（runtime 为 true，`runtimes/unified_exec.rs:166-168`）、`wants_no_sandbox_approval`（`Never/OnRequest` 的常规拒绝直接返回原 denial，`:367-391`）、`unsandboxed_allowed`（`:392-401`）；`bypass_retry_approval`（`:414-416`）判定是否免二次审批——首次已批过（`already_approved`）或策略为 `Never` 则不再问（`tools/sandboxing.rs:315-321`），否则以 retry_reason 再问一次用户（`:417-443`）。第二次尝试仅在 `unsandboxed_allowed` 时才 `SandboxType::None` 脱沙箱（`:445-461`）；再失败原样返回（`:505-517`）。

## 3. Go 翻译

只画控制面骨架（审批/选沙箱/重试的编排，不实现任何真实沙箱）：

```go
// 对应 ToolOrchestrator::run 的编排顺序：approval → select sandbox → attempt → retry。
func runTool(ctx context.Context, s *Session, cmd Command) (Output, error) {
    req := s.execPolicy.ApprovalRequirement(cmd) // Skip / NeedsApproval / Forbidden
    alreadyApproved := false
    switch req.Kind {
    case Forbidden:
        return Output{}, ErrRejected(req.Reason)
    case NeedsApproval, SkipIfStrict:
        if err := s.RequestApproval(ctx, cmd, req.Reason); err != nil {
            return Output{}, err // Denied→Rejected；Abort→中断 turn（见 turn-loop.md）
        }
        alreadyApproved = true
    }

    allowed, sandbox := selectInitialSandbox(cmd, s.Permissions) // 与审批结果无关
    out, err := attempt(ctx, cmd, sandbox)
    if err == nil || !isSandboxDenied(err) {
        return out, err
    }

    // SandboxDenied 分支：门控后可能二次审批 + 脱沙箱重跑一次。
    if !canEscalate(s.Permissions) { return out, err } // denied-read 等情况禁止脱沙箱
    if !(alreadyApproved || s.ApprovalPolicy == Never) {
        if err := s.RequestApproval(ctx, cmd, retryReason(out)); err != nil {
            return Output{}, err
        }
    }
    return attempt(ctx, cmd, SandboxNone)
}
```

与 Rust 的语义差异：Rust 的取消有两层（`parallel.rs:202-234` 的 dispatch select 与 `oneshot.rs:57-74` 的执行 select），而审批 oneshot 挂起本身不 select cancellation token，靠丢弃 sender 回落 Abort——Go 版本要显式设计这两层的取消传播；`ApprovedForSession` 缓存（`tools/sandboxing.rs:70-116`）在 Go 里对应一个带 key 的已批准 map，别把「没弹窗」误读成「没走审批」。

## 4. 易误解边界

- **审批在 spawn 前**：拒绝路径不产生进程；SandboxDenied 升级重试才可能出现「二次审批 + 二次 spawn」。
- **审批 ≠ 脱沙箱**：`already_approved` 只影响升级重试的二次审批（`orchestrator.rs:414-416`）；首次沙箱选择与审批结果无关。只有 exec policy 全信任规则或获批的提权请求才首次脱沙箱（`tools/sandboxing.rs:238-267`）。
- **取消有三层，别混**：dispatch 层 select（`parallel.rs:202-234`）、one-shot 执行层 select（`oneshot.rs:57-74`）、审批挂起靠 oneshot 关闭回落 Abort（`session/mod.rs:2875`）。
- **被沙箱的是工具子进程**：macOS 是 `sandbox-exec` 包原命令、Linux 是 `codex-linux-sandbox` helper、Windows 是 spawn 时施加受限 token；不能据此说 Codex 控制进程也在同一沙箱里（见 [sandbox-permissions.md](sandbox-permissions.md)）。
- **`ApprovedForSession` 缓存**：同 key 命令后续跳过弹窗（`tools/sandboxing.rs:70-116`）。
- **交互式 vs one-shot**：一个无完成超时、可续写 stdin；一个有 `timeout_ms` 且取消时终止进程（`exec_command.rs:441-447`、`oneshot.rs:26-86`）。

## 5. 没验证的

- 远程 exec-server 执行端的完整实现（`process_manager.rs:1261-1287` 只核对了分支入口）；Windows 受限 token 后端（`windows-sandbox-rs`）内部细节。
- `ExecServerEnvConfig`、shell snapshot、网络代理/网络审批（`deferred_network_approval`）的交互没有展开。
- `is_likely_sandbox_denied` 启发式的具体规则只确认了调用位置，没有逐条核对。
- 本文行号对应当前 `codex/` 这份源码；`codex-upstream/` 结构已分叉，行号不通用。

（以上均为阅读观察，非上游结论。）
