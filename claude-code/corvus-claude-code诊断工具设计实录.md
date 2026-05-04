# corvus — Claude Code 诊断工具设计实录

## 背景

Claude Code 接第三方 provider（中转服务、Bedrock）时，会遇到一类很隐蔽的错误：
请求本身完全合法，但 provider 返回 400/403/503，原因是模型名不兼容。
错误信息里看不出是"我发的 model 字段写错了"，只能靠猜。

这促使了 corvus 的诞生——一个专为 Claude Code + 第三方 provider 场景设计的诊断 CLI。

---

## 核心问题：三层命名不匹配

```
Claude Code 内部名 → 环境变量 → Provider 实际支持的 model ID
      "Opus 4"    →  claude-opus-4-20250514-thinking  →  price_not_configured
```

任何一层不匹配都会静默失败，报错和真实原因之间没有直接关联。
corvus 的价值就是把这三层映射关系可视化出来。

---

## sniff 的架构决策

### 为什么不需要 MITM 证书

一个关键认知：Claude Code → corvus → upstream 这条链路不需要 TLS 解密。

```
Claude Code ──HTTP──→ localhost:8080  ──HTTPS──→ upstream
              (plaintext，localhost 不需要 TLS)    (ureq/reqwest 自动)
```

Claude Code 侧用 HTTP 连本地代理，代理侧用 HTTPS 连上游。
不需要 MITM 证书，不需要 CONNECT 隧道，比 mitmproxy 的完整模式简单两个数量级。

mitmproxy 的 `--mode reverse:https://upstream` 和 corvus sniff 是完全相同的架构，
差别只在分析层：mitmproxy 通用，corvus 懂 Claude Code 的 model/token/beta flag 语义。

### 为什么 ureq 是错的

第一版用 `ureq`（同步阻塞）+ `spawn_blocking` 实现转发：

```rust
// 问题：等整个响应 buffer 完才返回
resp.into_reader().read_to_end(&mut body_buf)
```

Claude Code 几乎所有请求都是 `stream: true`（SSE），
这意味着每次请求 Claude Code 都会卡住等待所有 token 生成完，才突然收到全部内容。
功能上正确，用起来像卡死了。

**修复**：换 `reqwest`（async），用 `bytes_stream()` 直接 pipe 给 hyper 的 `StreamBody`：

```rust
// 修复后：每个 chunk 即时透传
let byte_stream = upstream_resp.bytes_stream()
    .map(|chunk| Ok::<_, Infallible>(Frame::data(chunk.unwrap_or_default())));
let body = BodyExt::boxed(StreamBody::new(byte_stream));
```

顺带去掉了 `spawn_blocking` 这个"同步转异步桥"。

### BoxBody error type 用 Infallible

流式响应一旦开始发送（header 已发出），如果中途出错无法回滚 HTTP 状态码。
唯一能做的是截断流。所以响应 body 的 error type 用 `Infallible`——错误在内部 log，不暴露给 hyper。

```rust
type RespBody = BoxBody<Bytes, Infallible>;
```

---

## launcher 模式的设计

### 核心限制

env var 是进程启动时的快照，运行中的进程无法被从外部修改。
所以"不关旧窗口就能抓到请求"在不动用 pfctl/iptables 的前提下是做不到的。

### env-wrapper 模式

参考 Unix 的 `env FOO=bar cmd` 模式，让 corvus 直接启动 Claude Code：

```bash
corvus sniff -- claude
```

关键实现细节：**先 bind 端口，再 spawn 子进程**，消除竞态：

```rust
let listener = rt.block_on(crate::proxy::server::bind_listener(port))?; // 1. 先绑端口（确认成功）
std::thread::spawn(move || { /* proxy accept loop */ });                 // 2. 再后台跑
// 子进程在此时启动，端口已确认就绪
let _ = std::process::Command::new(&launch[0])
    .env("ANTHROPIC_BASE_URL", format!("http://localhost:{}", port))
    .status()?;
```

实际工作流：发现报错 → Ctrl+C 旧的 → `corvus sniff -- claude` 一条命令重启。

---

## CI clippy 坑

`-D warnings` 会把所有 warning 变成 error。碰到的主要坑：

**uninlined_format_args**：`format!("{}", x)` 要改成 `format!("{x}")`，涉及 30+ 处。
`cargo fix` 只修 bin crate 根，module 文件里的不处理。
解法：在 `main.rs` 加 crate 级 allow，而不是逐处修复：

```rust
#![allow(clippy::uninlined_format_args)]
```

**dead_code**：deserialized-from-JSON 的字段（Settings、Permissions）
虽然"读入了"但没有被 Rust 代码访问，clippy 仍然报 dead_code。
用 `#[allow(dead_code)]` 标注结构体，而不是删字段（字段本身有存在价值）。

---

## 关键知识点

| 场景 | 结论 |
|------|------|
| SSE 流式代理 | 必须用 async HTTP client，同步 client 会 buffer 整个响应 |
| 拦截已运行进程 | 不可能（无 sudo），唯一方案是重启进程通过代理 |
| HTTPS 代理 vs 反向代理 | 反向代理不需要证书，MITM 才需要 |
| BoxBody error type | streaming 用 Infallible，错误只能截断不能上报 |
| bind-before-spawn | 端口先绑定再启动子进程，避免竞态 |
