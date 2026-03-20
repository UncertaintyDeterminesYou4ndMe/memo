# Claude Code API 错误排查：beta flag 与模型访问权限

## 背景

Claude Code 在使用第三方 provider（AWS Bedrock / 国内中转服务）时，连续遇到两个相关错误：

### 错误 1：invalid beta flag
```
API Error: 400 ValidationException: invalid beta flag
```

### 错误 2：无权访问模型
```
API Error: 403 该令牌无权访问模型 claude-sonnet-4-6
```

两个错误都指向同一个核心问题：**模型版本与 provider 支持的模型列表不匹配**。

---

## 核心问题分析

### 现象链

1. 用户切换模型到 `Opus 4.6 (1M context)`
2. 请求发送后，AWS Bedrock 返回 `invalid beta flag`
3. 切换到另一个 provider（apimart），返回 `无权访问模型 claude-sonnet-4-6`

### 关键判断

**Claude Code 的模型命名 ≠ Provider 的模型命名**

| Claude Code 显示 | 实际请求模型名 | Bedrock 模型 ID |
|-----------------|---------------|-----------------|
| Opus 4.6 | `claude-sonnet-4-6` ? | `anthropic.claude-3-opus-20240229-v1:0` |

Claude Code 内部可能使用了一些**非标准的模型别名**，这些别名在：
- **AWS Bedrock**：被 `anthropic-beta` header 标记为无效 beta flag
- **中转服务商**：直接返回 403，表示该模型 ID 不在服务范围内

---

## 排查步骤

### 1. 确认实际请求的模型 ID

```bash
CLAUDE_DEBUG=1 claude
# 然后发送任意消息，观察请求体中的 "model" 字段
```

### 2. 检查 Provider 支持的模型列表

**AWS Bedrock**：
- 控制台 → Amazon Bedrock → Model access
- 确认已启用 Claude 3 Opus / Sonnet 等模型
- 注意：Bedrock 的模型 ID 格式为 `anthropic.claude-3-xxx`

**中转服务（apimart 等）**：
- 查看服务商文档的「支持模型列表」
- 确认 `claude-sonnet-4-6` 或等效模型是否在列表中

### 3. 模型映射关系

| Anthropic 官方 | Bedrock 模型 ID | Claude Code 内部名 |
|---------------|-----------------|-------------------|
| claude-3-opus-20240229 | anthropic.claude-3-opus-20240229-v1:0 | opus / claude-opus |
| claude-3-sonnet-20240229 | anthropic.claude-3-sonnet-20240229-v1:0 | sonnet / claude-sonnet |
| claude-3-haiku-20240307 | anthropic.claude-3-haiku-20240307-v1:0 | haiku / claude-haiku |

**注意**：`claude-sonnet-4-6` 这个命名不是标准格式，可能是 Claude Code 的内部别名或版本标记。

---

## 解决方案

### 方案 1：使用标准模型名（推荐）

在 Claude Code 中明确选择标准模型，避免使用带版本号的别名：

```
/model
# 选择 "Sonnet 4" 或 "Opus 4" 而不是 "Sonnet 4.6" 等带小版本的选项
```

### 方案 2：更换 Provider

如果当前 provider 不支持所需模型：

```bash
# 配置新的 provider
claude config set apiKey <your_key>
claude config set apiEndpoint <endpoint_url>
```

### 方案 3：降级到稳定版本

```
/model
# 选择 "Sonnet 3.5" 或 "Haiku" 等更广泛支持的模型
```

---

## 调试工具总结

| 场景 | 工具 | 命令 |
|------|------|------|
| 查看实际请求 | DEBUG 模式 | `CLAUDE_DEBUG=1 claude` |
| 抓包分析 | tcpdump | `sudo tcpdump -i en0 -w capture.pcap port 443` |
| 检查配置 | 查看 config | `cat ~/.claude.json \| jq '.projects["'$(pwd)'"]']` |

---

## 结论

两个错误的根本原因相同：**Claude Code 的模型别名与 Provider 支持的模型 ID 不匹配**。

- `400 invalid beta flag` = Bedrock 不认识这个模型别名，将其误判为 beta flag
- `403 无权访问` = 中转服务直接拒绝未知的模型 ID

**最佳实践**：使用 `/model` 命令选择标准模型名，避免手动输入或选择带小版本号（如 4.6）的模型选项。
