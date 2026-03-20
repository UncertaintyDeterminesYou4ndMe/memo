# Claude Code 抓包调试与 "invalid beta flag" 错误分析

## 背景

Claude Code 在使用 AWS Bedrock 作为 provider 时，出现以下错误：

```
API Error: 400 ValidationException: invalid beta flag
```

错误发生在调用 `InvokeModelWithResponseStream` API 时，需要定位具体请求内容以诊断问题。

---

## 核心问题：HTTPS 抓包的局限性

最初尝试用 `tcpdump` 抓包分析请求：

```bash
sudo tcpdump -i en0 -n -s 0 -w capture.pcap 'host 10.20.11.91 and port 443'
```

**关键判断**：tcpdump 能抓到 TLS 握手和目标 IP，但 **HTTPS payload 是加密的**，无法直接查看请求体中的 headers 和 body。对于调试 API 错误（尤其是 header 相关的问题），抓包的价值有限。

---

## 更优方案：CLAUDE_DEBUG 环境变量

与其抓包解密，不如直接启用 Claude Code 的调试模式：

```bash
export CLAUDE_DEBUG=1
export ANTHROPIC_DEBUG=1
claude
```

这样可以在终端直接看到完整的请求/响应内容，包括所有 headers 和 body，无需处理 TLS 加密。

---

## "invalid beta flag" 错误分析

从错误信息可以推断：

| 观察 | 推断 |
|------|------|
| `InvokeModelWithResponseStream` | Bedrock 的流式调用 API |
| `ValidationException` | 请求参数验证失败 |
| `invalid beta flag` | `anthropic-beta` header 包含 Bedrock 不支持的值 |

**可能原因**：
1. Claude Code 发送了某些 Anthropic 原生 API 支持的 beta flags，但 Bedrock 不支持
2. 模型版本（Opus 4.6）与 beta flags 的组合在 Bedrock 侧被拒绝

**快速修复尝试**：
```bash
rm ~/.claude/config.json 2>/dev/null  # 清除可能的问题配置
CLAUDE_DEBUG=1 claude                  # 重新启动并观察调试输出
```

---

## 调试工具对比

| 工具 | 适用场景 | 局限 |
|------|----------|------|
| `tcpdump` | 网络层问题、连接问题 | 无法解密 HTTPS payload |
| `CLAUDE_DEBUG=1` | API 请求内容调试 | 需要重启 Claude Code |
| `mitmproxy` | 需要拦截修改 HTTPS 请求 | 需要安装证书、配置代理 |

**结论**：CLI 工具的 API 调试，优先使用内置的 DEBUG 环境变量，抓包是最后手段。
