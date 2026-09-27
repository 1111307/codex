# codex 的 MCP 拓扑：客户端 + 一个反向服务器

围绕三个问题：codex 说什么 MCP 方言、用什么传输、扮演什么角色。全部结论来自 fork 基线实读。

## 1. SDK 与协议版本：官方 rmcp，钉死 3.2.0

```toml
rmcp = { version = "=3.2.0", default-features = false }
```

（codex-rs/Cargo.toml:430）`=` 前缀是精确钉版——MCP 协议还在快速演进，官方 SDK 的 minor 升级即可能有破坏性改动，钉死是防御性选择。「观察」：3.x 的 rmcp 连 trait 签名都在变，这类拉取型协议客户端，锁版本比追新重要。

协议版本协商在 `codex-rs/rmcp-client/src/protocol_mode.rs:9-37`：

```rust
pub enum McpProtocolMode {
    Legacy,
    V20260728,
}

// :21-22（试探顺序）
Self::Legacy => ProtocolVersion::V_2025_06_18,
Self::V20260728 => ProtocolVersion::V_2026_07_28,
```

两个模式对应官方 SDK 的两代握手语：**Legacy** 模式走旧的 `initialize` 流程（`V_2025_06_18`）；**V20260728** 模式（:30-31）优先提议 `V_2026_07_28`、失败回落 legacy——即服务器说哪种方言，客户端就切哪种。

## 2. 传输：只有 Stdio 和 StreamableHttp 两种

`codex-rs/config/src/mcp_types.rs:554-568` 的 transport enum：

```rust
pub enum McpServerTransportConfig {
    /// Launch the MCP server as a subprocess, communicating over stdin/stdout.
    Stdio { command: String, args: ..., env: ..., ... },
    /// Connect to an MCP server over HTTP using the Streamable HTTP transport.
    StreamableHttp { url: String, ... },
}
```

没有 SSE 传输（grep 全仓无 `SseTransport` 等价物）。子进程用 stdio，HTTP 用 Streamable HTTP——这是 MCP 官方目前主推的两条传输，旧 HTTP+SSE 被有意省略了。「观察」：新型服务器侧主要在 sse→streamable-http 迁移，codex 只保留目标态,而非兼容中间态。

运行时的并发初始化与超时控制见 `rmcp-client/src/rmcp_client.rs`（PendingTransport 路径，pending map + oneshot，模式与 session 审批挂起同款）。

## 3. 角色反转：TUI 开了一个反向 MCP 服务器

codex 的核心进程在整个 MCP 拓扑里**默认是 client**（`LoggingClientHandler` 只是日志透传)——外部 MCP server 提供工具,codex 调用它们。

但有一个精心设计的例外：TUI 前端会启动一个**自己的 server**，把 TUI 侧持有的「动态工具」暴露给核心进程。`tui/src/dynamic_tools_mcp.rs:114`：

```rust
let listener = TcpListener::bind("127.0.0.1:0").await?;
```

关键细节：

- **`127.0.0.1:0`**：只绑本地回环 + 让 OS 分配随机端口——不监听公网，不碰固定端口；
- **AUTHORIZATION 中间件**（`dynamic_tools_mcp.rs:10`, `:189`）：即使只听 localhost 也校验请求头,防止本机其他进程直连冒充 TUI；
- **审批门控后才暴露工具**：对应 `state/turn.rs` 的 `pending_dynamic_tools` 表——核心收到一个「这个 server 有你要的工具」的信号,真正调用仍走回合内审批链。

「观察」：这解决的是「前端持有的工具如何注入核心对话」问题。换个角度：codex 的工具执行全在核心,权限模型挂在核心,但有些工具（如 IDE 侧快捷操作）宿主注定在前端——与其造一个私有协议,不如反转一下 MCP 的 client/server 方向,复用现有的序列化和版本协商。MCP 在这里被当成**通用的「跨进程调用带认证」框架**用,而不是「外部工具市场」。

## 4. CLI 子命令：纯管理面，没有连接测试

`cli/src/mcp_cmd.rs` 提供 list / get / add / remove / login / logout,全部只读写 `config.toml`,不发起连接。「观察」：没有 `mcp test`/`mcp ping` 之类的连通性诊断命令——要么没做到爱吃,要么在 TUI/启动日志里体现,CLI 刻意保持纯配置管理。

## 5. 汇总

| 维度 | codex 的选择 | 备注 |
|---|---|---|
| SDK | 官方 rmcp，`=3.2.0` 精确钉版 | 防御协议快速演进 |
| 协议版本 | Legacy / V20260728 双模式 + 回落 | protocol_mode.rs:21-31 |
| 传输 | Stdio + StreamableHttp，**无 SSE** | mcp_types.rs:554-568 |
| 角色 | 核心=client；唯一例外是 TUI 的反向 server | 动态工具通道 |
| 安全 | Server 侧 127.0.0.1:0 + AUTHORIZATION 头校验 | 本机回连也要认证 |
| CLI | 纯配置管理面，无连通性测试 | mcp_cmd.rs |

「观察」与「上游结论」的区分：钉版理由、无 SSE 的动机、无 ping 命令的取舍均为推断，标为「观察」；传输枚举、版本列表、端点绑定、AUTHORIZATION 使用为逐字实读。
