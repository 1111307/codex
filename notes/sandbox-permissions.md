# 沙箱权限：进程裸奔，子进程坐牢

围绕一个问题：codex 启动后到底拿到了什么权限？结论先行——**codex 进程自己不在沙箱里,被沙箱的只有它 exec 出去的子进程**。

## 1. 三档模式与默认值

`codex-rs/protocol/src/config_types.rs:104-118`（`#[serde(rename_all = "kebab-case")]`）：

```rust
pub enum SandboxMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
}
```

默认值在 `config/src/config_toml.rs:756-789` 的 `derive_permission_profile`：

- 用户显式配置 `sandbox_mode` → 用它；
- 未配置但该目录有信任决策（trusted **或** untrusted）→ `WorkspaceWrite`;
- 两个都没有 → `unwrap_or_default()` = `ReadOnly`（config_types.rs 的 `#[default]`）;
- Windows 且沙箱 disabled → `WorkspaceWrite` 强制降级 `ReadOnly`（:777-786 的注释原话：「If no sandbox_mode is set but this directory has a trust decision, default to workspace-write except on unsandboxed Windows where we default to read-only.」）。

## 2. 读权限：所有模式都是全盘可读

「沙箱 = 工作目录权限」是常见误解。`protocol/src/permissions.rs:594-601`：

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

`Root` = `/`，单条目、Read——**最保守的 ReadOnly 模式就是「整个根目录只读」**。`Default for FileSystemSandboxPolicy` 就是它（:604-607）。

`WorkspaceWrite` 也一样,`permissions.rs:799-810`：底座先放一条 `Root/Read`,再叠 writable_roots。**读从不收紧,只有写有梯度**。写梯度：

| 模式 | 可写范围 |
|---|---|
| ReadOnly | 无（全盘只读） |
| WorkspaceWrite | cwd（project_roots）+ /tmp + $TMPDIR + writable_roots，**剔除** `.git` / `.agents` / `.codex` 等保护路径 |
| DangerFullAccess | 无沙箱 |

可写判定实现在 `protocol/src/protocol.rs:1140-1146`（`WritableRoot::is_path_writable`），Exact / BelowRoot / WithinRootPrefix 三种匹配语义。

## 3. 沙箱的真正对象：exec 出去的子进程

codex 的 Rust 进程自身不进沙箱。拿 Linux 说话,`linux-sandbox/src/bwrap.rs:386` 的注释和 `:480-560` 的 mount 策略：

```text
bwrap 参数：--ro-bind / /
```

（bwrap.rs:386-390 一带,整根只读挂载后,按 policy 对 writable_roots 开 tee/bind 例外。）`LINUX_PLATFORM_DEFAULT_READ_ROOTS`（:46）列出的是平台默认放行的只读根列表,不是「仅这些可读」。

macOS 侧 `sandboxing/src/seatbelt_base_policy.sbpl`（:7）：

```scheme
; start with closed-by-default
(deny default)
```

closed-by-default,再按 policy 放行——citing 里写着灵感来自 Chrome 的 renderer sandbox（:3-5 注释给了 chromium 源码链接）。

**沙箱边界图**：

```
codex 进程（Rust, 不受沙箱约束,以你的用户身份跑）
   │ exec + bwrap/Seatbelt 参数
   ▼
工具子进程（真正被 --ro-bind / / 或 (deny default) 罩住的那个）
```

推论：codex 自身的文件读写、网络、配置加载全按你的用户权限走；「沙箱」这一层只承诺「**模型通过工具发起的副作用**被拦在策略内」。这是「能力约束」而不是「进程隔离」。

## 4. Go 翻译

| Rust | Go | 注意 |
|---|---|---|
| `FileSystemSpecialPath::Root` + `AccessMode::Read` | `map[string]bool`（path→writable）,或 `-ro` 参数表 | 策略是数据，别写成 if 链 |
| derive_permission_profile 的三层 fallback | `resolveMode(explicit, trust, default)` 单函数 | Windows 降级逻辑收敛在一处,别散 |
| `WritableRoot::is_path_writable` 3 语义 | `filepath.Clean` + `strings.HasPrefix`（注意 `/` 绑尾才算 below-root） | 路径匹配的边界 case 全在这里 |
| bwrap 拼参数 | `exec.Command("bwrap", ...)` | 价格一样：进程池为空即 cold-start |
| policy 文件数据化（.sbpl / entries vec） | 表驱动 | 读 Windows 降级细则时别硬编码 |

「观察」：codex 没有像部分 harness 那样给「全进程 MAC/容器隔离」的选项——selinux/apparmor/docker 都不依赖。信任模型里「宿主机管理员/用户」始终在圈外,圈住的是「模型驱动的工具」。

## 5. 没验证的上游细节

- `LINUX_PLATFORM_DEFAULT_READ_ROOTS` 的完整列表内容（只确认了它在 :46 及被 :501 引用），没有逐项核对与解释。
- Windows 侧「沙箱 disabled 时降 ReadOnly」的具体检测（`WindowsSandboxLevel::Disabled` 的来源/开关名），未实读其配置路径。
- `EXTERNAL` sandbox（Linux 下另一种模式）的完整语义,只在别的笔记里侧面提到,未定位实现文件。
