# 逆向工程 Claude Code System Prompt

## 背景

通过将 Claude Code 指向 Moonshot API（`kimi-k2.5` 模型），以交互方式逆向获取其 System Prompt 的核心内容，包括工具定义、行为规则、编码哲学等。

---

## 核心发现：API 替换的可行性

Claude Code CLI 可通过环境变量指向任意兼容 Anthropic API 格式的端点：

```bash
export ANTHROPIC_BASE_URL="<YOUR_BASE_URL>"
export ANTHROPIC_AUTH_TOKEN="<YOUR_API_KEY>"
export ANTHROPIC_MODEL="kimi-k2.5"
claude -p
```

这意味着 **Claude Code 的行为主要由客户端（CLI 工具）的 System Prompt 定义**，而非服务端模型。模型仅负责理解和执行这些指令。

---

## System Prompt 核心结构

### 1. 身份定义（默认 vs 角色覆盖）

**默认身份：**
```
I'm Claude, an AI assistant created by Anthropic.
Purpose: Software engineering assistance
Values: Safe, correct code; avoid over-engineering; be concise
```

**角色覆盖机制：**
- `CLAUDE.md` / `CLAUDE.local.md` 按优先级加载（全局 → 项目级）
- 后加载的文件覆盖前者的冲突指令
- 文件被监听，修改后自动重载

对比发现：排除 `CLAUDE.local.md`（Linus Torvalds 角色）后，问候语从 "你好，我是 Linus" 变为 "Hello! How can I help you?"，编码哲学从 "好品味/Never break userspace" 变为 "Pragmatic and safety-focused"。

---

### 2. 工具定义（25+ 工具）

**核心原则：专用工具优先于 Bash**

| 工具 | 用途 | 关键约束 |
|------|------|----------|
| `Read` | 读取文件 | 支持 offset/limit 分块读取大文件 |
| `Edit` | 精确字符串替换 | 必须先 Read，old_string 必须精确匹配 |
| `Write` | 创建/覆盖文件 | 破坏性操作，谨慎使用 |
| `Glob` | 文件模式匹配 | 避免 `**` 在根目录（太宽泛）|
| `Grep` | 内容搜索 | 基于 ripgrep，支持 regex 和上下文 |
| `Agent` | 子代理 | 支持 `Explore`/`Plan`/`general-purpose` 三种类型 |
| `TodoWrite` | 任务管理 | 一次只能有一个 `in_progress` 任务 |

**Agent 类型设计：**
- `Explore`：只读工具，用于代码库探索（不修改代码）
- `Plan`：只读工具，用于架构规划
- `general-purpose`：完整工具访问，用于实现

---

### 3. 编码哲学

**默认风格：**
- Minimal changes — 只修改直接请求的内容
- Trust internal code — 不在内部代码边界加防御
- No backwards-compatibility hacks — 直接改，不 shim
- Skip docstrings for unchanged code — 不污染未修改部分

**Linus 角色覆盖的风格：**
- "Good Taste" — 重写使特殊情况消失于正常情况
- "Never break userspace" — 向后兼容性神圣
- 超过 3 层缩进 = 重设计
- 函数短小，只做一件事

**关键洞察：** 编码风格通过 `CLAUDE.md` 可高度定制，但工具使用规则是硬编码在 System Prompt 中的。

---

### 4. 安全规则

**必须确认的操作：**
- `rm -rf`, `DROP TABLE`, `git push --force`
- 修改 CI/CD、创建/关闭 PR
- 向外部服务发送消息

**Git 安全协议：**
- 绝不更新 git config
- 绝不 skip hooks（除非用户明确要求）
- 绝不 force push 到 main
- 总是创建新 commit，而非 amend

---

### 5. 响应风格规则

**Conciseness（简洁性）：**
- One sentence beats three
- Lead with answers, not reasoning
- No filler phrases ("Let me...", "I think...")
- No trailing summaries of what just did
- 代码引用格式：`file_path:line_number`

**Emoji：** 除非用户明确要求，否则不使用。

---

## 实验：排除角色文件的影响

将 `CLAUDE.local.md` 备份并移除后，观察到：

| 维度 | 有 Linus 角色 | 默认行为 |
|------|---------------|----------|
| 问候 | "你好。我是 Linus。" | "Hello! How can I help you?" |
| 哲学 | 好品味、永不破坏 userspace | Pragmatic and safety-focused |
| 沟通 | 犀利、零废话 | Direct and action-oriented |
| 代码审查 | 品味评分 🟢🟡🔴 | 标准安全检查 |

**结论：** `CLAUDE.md` 机制非常强大，可以完全改变 AI 的行为风格而不影响工具使用能力。

---

## 逆向工程的方法论

1. **环境替换法**：将 API 端点指向可控的替代服务（如 Moonshot）
2. **直接询问法**：询问 "List all available tools"、"What are your coding guidelines?" 等
3. **对比实验法**：通过添加/移除 `CLAUDE.md` 观察行为变化

---

## 设计哲学总结

Claude Code 的 System Prompt 体现了几点精巧设计：

1. **分层指令**：基础行为（硬编码）+ 角色覆盖（`CLAUDE.md` 可配置）
2. **工具优先**：强制使用专用工具而非 Bash，确保可预测性和安全性
3. **上下文隔离**：Sub-agent 完全隔离，必须通过 prompt 传递所有上下文
4. **渐进式授权**：权限模式从保守（default）到激进（bypassPermissions）渐进

> 逆向工程的价值不在于复制，而在于理解"为什么这样设计"。
