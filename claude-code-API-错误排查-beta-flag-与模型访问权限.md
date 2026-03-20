# Claude Code API 错误排查：beta flag 与模型访问权限

## 背景

Claude Code 在使用第三方 provider（AWS Bedrock / 国内中转服务）时，连续遇到多个相关错误：

### 错误 1：invalid beta flag
```
API Error: 400 ValidationException: invalid beta flag
```

### 错误 2：无权访问模型
```
API Error: 403 该令牌无权访问模型 claude-sonnet-4-6
```

### 错误 3：模型价格未配置（new_api）
```
API Error: 500 模型 claude-opus-4-20250514-thinking 倍率或价格未配置，请联系管理员设置或开始自用模式
```

---

## 核心问题分析

### 环境变量配置

```bash
export ANTHROPIC_BASE_URL="<YOUR_BASE_URL>"
export ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-4-5-20251001
export ANTHROPIC_DEFAULT_SONNET_MODEL=claude-sonnet-4-6
export ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-4-20250514-thinking
export ANTHROPIC_API_KEY="<YOUR_API_KEY>"
```

### 关键判断

**Claude Code 的模型命名 ≠ Provider 支持的模型命名 ≠ 中转服务的定价配置**

三个层级的不匹配：

```
Claude Code 内部 → 环境变量映射 → 中转服务配置 → 实际模型
     ↓                ↓              ↓            ↓
   "Opus 4"    →  claude-opus-4-20250514-thinking  →  未配置价格 → 报错
```

---

## 错误分类与原因

| 错误 | Provider | 根本原因 |
|------|----------|----------|
| `400 invalid beta flag` | AWS Bedrock | 模型名被当成非法 beta flag |
| `403 无权访问模型` | apimart | 模型 ID 不在服务支持列表 |
| `500 倍率或价格未配置` | new_api | 模型未在后台配置定价 |

---

## 解决方案

### 方案 1：使用中转服务支持的标准模型名

查看服务商文档，使用他们实际支持的模型 ID：

```bash
# 改为服务支持的标准模型名
export ANTHROPIC_DEFAULT_OPUS_MODEL="claude-3-opus-20240229"
export ANTHROPIC_DEFAULT_SONNET_MODEL="claude-3-sonnet-20240229"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="claude-3-haiku-20240307"
```

### 方案 2：去掉特殊后缀

`-thinking` 后缀是 Anthropic 的实验性功能，多数中转服务不支持：

```bash
# 错误
export ANTHROPIC_DEFAULT_OPUS_MODEL="claude-opus-4-20250514-thinking"

# 正确
export ANTHROPIC_DEFAULT_OPUS_MODEL="claude-opus-4-20250514"
```

### 方案 3：直接在 Claude Code 内切换模型

```
/model
# 选择标准模型，不依赖环境变量映射
```

### 方案 4：联系管理员配置（如果是自建中转）

如果是 new_api / OneAPI 等中转平台，需要在后台添加模型定价：
- 渠道管理 → 编辑 → 模型列表中添加对应模型
- 倍率设置 → 配置该模型的计费倍率

---

## 模型命名映射参考

| Anthropic 官方 | 常用中转服务支持名 | Claude Code 环境变量 |
|---------------|-------------------|---------------------|
| claude-3-opus-20240229 | claude-3-opus / claude-opus | ANTHROPIC_DEFAULT_OPUS_MODEL |
| claude-3-sonnet-20240229 | claude-3-sonnet / claude-sonnet | ANTHROPIC_DEFAULT_SONNET_MODEL |
| claude-3-haiku-20240307 | claude-3-haiku / claude-haiku | ANTHROPIC_DEFAULT_HAIKU_MODEL |
| claude-3-5-sonnet-20241022 | claude-3-5-sonnet | ANTHROPIC_DEFAULT_SONNET_MODEL |

**注意**：`-thinking`、`-4-6`、`-4-5-20251001` 等后缀多为非标准命名，需要确认服务商是否支持。

---

## 排查步骤

### 1. 确认实际请求的模型 ID

```bash
CLAUDE_DEBUG=1 claude
# 发送任意消息，观察请求体中的 "model" 字段
```

### 2. 检查 Provider 支持的模型列表

**AWS Bedrock**：
- 控制台 → Amazon Bedrock → Model access
- 模型 ID 格式：`anthropic.claude-3-opus-20240229-v1:0`

**中转服务（new_api/OneAPI）**：
- 管理后台 → 渠道 → 查看支持的模型列表
- 或者通过 API 测试模型可用性

### 3. 测试模型可用性

```bash
curl <YOUR_BASE_URL>/v1/models \
  -H "Authorization: Bearer <YOUR_API_KEY>"
```

---

## 调试工具总结

| 场景 | 工具 | 命令 |
|------|------|------|
| 查看实际请求 | DEBUG 模式 | `CLAUDE_DEBUG=1 claude` |
| 测试 API 连通性 | curl | `curl <YOUR_BASE_URL>/v1/models` |
| 检查配置 | 查看 config | `cat ~/.claude.json \| jq '.projects["'$(pwd)'"]'` |

---

## 结论

三个错误的根本原因相同：**模型命名在三个层级间不匹配**。

- `400 invalid beta flag` = Bedrock 不认识自定义模型名
- `403 无权访问` = 中转服务模型列表不包含该模型  
- `500 价格未配置` = 中转服务支持该模型但未配置计费倍率

**最佳实践**：
1. 使用服务商文档中明确列出的模型 ID
2. 避免使用带 `-thinking`、`-4-6` 等非标准后缀的模型名
3. 用 `CLAUDE_DEBUG=1` 确认实际发送的模型名
