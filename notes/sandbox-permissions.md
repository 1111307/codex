# 沙箱权限：控制进程与工具子进程的边界

围绕一个问题：Codex 启动后到底拿到了什么权限？结论先行——**Codex Rust 控制进程本身通常不在它为工具搭建的沙箱内；执行工具启动的本地 OS 子进程按当前权限策略选择沙箱**。子 agent 的创建只是新建会话，不会因此直接起沙箱进程。完整时序见 [loops-agents-sandbox-go.md](loops-agents-sandbox-go.md)。

## 1. 三档模式与默认值

`codex-rs/protocol/src/config_types.rs:104-118`（`#[serde(rename_all = "kebab-case")]`）：

```rust
pub enum SandboxMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
}
```

以下**只针对 legacy `sandbox_mode` 推导路径**：`config/src/config_toml.rs:751-755` 特别要求先排除 `default_permissions`，命名的 `[permissions]` profile 应走新的编译管线，不能用这些三档模式反推。legacy 推导在 :756-828：

- 用户显式配置 `sandbox_mode` → 用它（但下面的 Windows 降级仍可作用于 WorkspaceWrite）；
- 未配置但该目录有信任决策（trusted **或** untrusted）→ `WorkspaceWrite`；
- 两个都没有 → `unwrap_or_default()` = `ReadOnly`（`protocol/src/config_types.rs:104-113` 的 `#[default]`）；
- Windows 且 sandbox level 为 `Disabled` → `WorkspaceWrite` 最终降为 `ReadOnly`（:782-790），包括显式选择的 WorkspaceWrite。

## 2. 读权限：legacy 预设全盘读 ≠ 任意 profile 全盘读

「沙箱 = 只能看工作目录」是常见误解，但原笔记的「所有模式全盘可读」也不成立。legacy ReadOnly 的文件系统基础规则见 `protocol/src/permissions.rs:594-601`：

```rust
fn read_only_file_system_entries() -> Vec<FileSystemSandboxEntry> {
    vec![FileSystemSandboxEntry::new(
        FileSystemPath::Special {
            value: FileSystemSpecialPath::Root,
        },
        FileSystemAccessMode::Read,
    )]
}
```

`Root` = `/`（Linux/macOS 文件系统语义），单条目 Read。**legacy ReadOnly 预设**采用这条全盘读基础规则；`Default for FileSystemSandboxPolicy` 也是它（:603-611）。

**legacy WorkspaceWrite 预设**也先放 `Root/Read`（`permissions.rs:799-810`），再叠 project roots、临时目录与指定可写路径。但显式文件系统 policy 支持更窄的可读根和 `FileSystemAccessMode::Deny`（`:676-687`）；`has_full_disk_read_access` 检查根级可读且没有被拒绝读的例外（`:891-900`）。**不能推断「读从不收紧」**。这些 legacy 预设的写梯度：

| 模式 | 可写范围 |
|---|---|
| ReadOnly | legacy 预设：根级读、无文件系统写权限 |
| WorkspaceWrite | legacy 预设：project_roots + /tmp + $TMPDIR + writable_roots（可配置例外）；`.git` / `.agents` / `.codex` 等 project 子路径默认只读（`permissions.rs:811-853`） |
| DangerFullAccess | legacy 映射为 `PermissionProfile::Disabled`（`config_toml.rs:815`），通常不请求本地平台沙箱 |

可写判定实现在 `protocol/src/protocol.rs:1140-1146`（`WritableRoot::is_path_writable`），Exact / BelowRoot / WithinRootPrefix 三种匹配语义。

## 3. 沙箱的真正对象：exec 出去的子进程

Codex 的 Rust 控制进程自身不因这条工具执行链被包入沙箱。Linux 的 bwrap 文件系统挂载实现位于 `linux-sandbox/src/bwrap.rs:382-397, 471-539`：**全盘读策略**用 `--ro-bind / /`；**受限读策略**若没有显式读取 `/`，先用 `--tmpfs /` 再把获准可读目录逐一 `--ro-bind`；有 denied-read carveout 时还会应用相应遮罩。`LINUX_PLATFORM_DEFAULT_READ_ROOTS` 在允许 include_platform_defaults 的受限读分支补充平台所需路径（`:499-505`），不能把它当成「所有配置天然全盘可读」的依据。

macOS 侧 `sandboxing/src/seatbelt_base_policy.sbpl`（:7）：

```scheme
; start with closed-by-default
(deny default)
```

closed-by-default，再按 policy 放行；文件开头的注释（:3-5）引用了 Chromium 的 sandbox 策略。

**沙箱边界图**：

```
Codex 控制进程（Rust：管理 session、模型请求、工具调度）
   │ spawn_agent → 新的 Session/turn task（仍在同一进程，继承当前权限配置）
   │ exec_command → approval → select sandbox → transform command
   ▼
本地命令 OS 子进程（按本次 attempt 的策略使用 bwrap/Seatbelt/Windows RestrictedToken，或 None）
```

源码事实：`core/src/tools/orchestrator.rs:141-312` 先审批、再选择第一次沙箱；`:322-489` 仅特定 SandboxDenied 可尝试升级。`core/src/tools/sandboxing.rs:444-529` 本地包装命令、远程下发权限；`core/src/unified_exec/process_manager.rs:1209-1213, 1261-1286, 1344-1359` 分别走 exec-server 或本地 spawn。`core/src/exec.rs:885-887` 明确底层 `exec` 不负责构建沙箱。**观察：**这条工具沙箱链约束的是工具所启动进程的能力，不能据此声称整个 Codex 宿主进程也被同一策略隔离；远端执行端另行施加其收到的权限。

## 4. Go 翻译

| Rust | Go | 注意 |
|---|---|---|
| `FileSystemSandboxPolicy.entries`（Read/Write/Deny） | 规则列表 + 平台后端 | 保留规则的例外与优先级，不能缩成 path→writable 的 bool map |
| legacy `derive_permission_profile` | `resolveLegacyMode(explicit, trust, windowsLevel)` | 命名 permissions profile 应走另一条编译路径 |
| `WritableRoot::is_path_writable` | 规范化路径后按路径段匹配 | `strings.HasPrefix` 单独使用不安全：前缀和 symlink/解析边界需处理 |
| `SandboxManager::transform` / `spawn_process` | OS 专用沙箱构造器 + `exec.CommandContext` | `cmd.Dir = cwd` 只改变工作目录，**不构成沙箱**；Windows 还需受限 token 后端 |
| 工具审批 / 沙箱选择 | 两个独立决策步骤 | 审批通过并不自动绕过沙箱；拒绝也不自动裸跑重试 |

「观察」：这条代码路径没有对宿主 Codex 控制进程本身套用同一个工具沙箱；不能由此断言所有宿主部署形态都不可能另有容器/MAC 限制。Go 标准库也没有跨 Linux/macOS/Windows 通用的强制文件系统沙箱 API，需要专门的 OS 后端。

## 5. 没验证的上游细节

- `LINUX_PLATFORM_DEFAULT_READ_ROOTS` 的完整列表内容（只确认了它被 `bwrap.rs:499-505` 的受限读分支引用），没有逐项核对与解释。
- Windows 侧「沙箱 disabled 时降 ReadOnly」的具体检测（`WindowsSandboxLevel::Disabled` 的来源/开关名），未实读其配置路径。
- `ExternalSandbox` / `FileSystemSandboxKind::ExternalSandbox` 的完整语义未逐条核对，不纳入上述 legacy 三档行为的概括。
