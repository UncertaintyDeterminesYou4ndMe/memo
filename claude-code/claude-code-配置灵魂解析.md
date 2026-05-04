# Claude Code 配置灵魂：~/.claude.json 深度解析

## 核心认知

`~/.claude.json` 不是一个"配置文件"，而是 Claude Code 的**用户状态数据库**。
它混合了四种性质完全不同的内容：偏好设置、A/B 实验开关、每个项目的安全授权状态、会话性能快照。

**一个关键判断**：这个文件**绝大多数字段不应该手动编辑**。
唯二适合手动干预的字段是：
- `projects.<path>.mcpServers` — 为某个项目配置 local 作用域的 MCP 服务器
- `projects.<path>.allowedTools` — 预授权工具，省去每次的交互确认

其他字段是程序自动维护的运行时状态，手动改了可能会被下一次启动覆盖。

---

## 文件结构分层

### 一、全局身份与偏好（顶层字段）

- `numStartups`：启动次数，是判断自己使用深度的直观指标
- `installMethod`：`"native"` / `"npm"` / `"homebrew"`，区分安装方式
- `theme`：`"dark-daltonized"` 是色盲友好的暗色主题
- `firstStartTime`：第一次使用时间（UTC）
- `userID`：SHA256 匿名 ID，用于 telemetry，不含个人信息
- `tipsHistory`：记录每个引导提示是在第几次启动时展示的，说明提示系统是基于启动次数触发的，不是随机的

### 二、Feature Flags — 真正的灵魂

Claude Code 内部代号是 **tengu**（天狗），所有实验性功能 flag 都以 `tengu_` 开头。

文件里有**两套并行的 A/B 测试系统**：

| 系统 | 字段 | 特点 |
|---|---|---|
| Statsig | `cachedStatsigGates` | 较早的系统，字段数量少，用于快速 flag |
| GrowthBook | `cachedGrowthBookFeatures` | 主力系统，~100 个 flag，控制大多数行为 |

两套系统对同一个 flag 可能给出不同的值（例如 `tengu_sumi` 在 Statsig 是 `false`，在 GrowthBook 是 `true`），说明**两套系统的判断逻辑相互独立**，GrowthBook 优先级更高。

**值得关注的非 boolean flag（字符串型）**：

- `tengu_swann_brevity: "focused"` — 控制回答的简洁程度，借用了普鲁斯特小说里"斯万"这个角色名作为代号，语义是"精练"
- `tengu_plank_river_frost: "user_intent"` — 意图识别模式
- `tengu_auto_mode_config: { enabled: "disabled" }` — 自动模式的开关，可能是一个三态开关而非 boolean

**推测几个有趣 flag 的含义**：

| Flag | 推测 |
|---|---|
| `tengu_penguins_enabled` | Background Agents（后台智能体）|
| `tengu_kairos_cron` | Kairos = 希腊语"时机"，定时/计划任务 |
| `tengu_bash_haiku_prefetch` | 用 Haiku 预判 bash 命令，加速响应 |
| `tengu_pid_based_version_locking` | 多实例并发时通过 PID 锁定版本稳定性 |
| `tengu_thinkback` | 可能是"回溯思考"，类似思维链的增强功能 |
| `tengu_session_memory` | 跨 session 的记忆持久化，尚未对该用户开放 |

这些 flag 大部分对普通用户不可见、不可配置，但通过观察它们的命名可以推断 Anthropic 正在探索的方向。

### 三、Projects — 使用足迹

每一个打开过 Claude Code 的目录都会生成一条记录，包含：

- **安全状态**：`hasTrustDialogAccepted`、`hasClaudeMdExternalIncludesApproved`
- **项目感知**：`exampleFiles` — 自动分析的代表性文件列表，用于上下文初始化
- **上次 session 快照**：花费、token 消耗、行数变更、各模型用量明细
- **性能指标**：`lastSessionMetrics` 包含帧渲染时长的 p50/p95/p99，用于 TUI 性能监控

**Token 缓存效率可以从 `lastTotalCacheReadInputTokens / lastTotalCacheCreationInputTokens` 比值推算**。比值越高，说明对同一上下文做了越多的连续深度迭代。

### 四、全局状态字段（散落在顶层）

- `githubRepoPaths`：GitHub 仓库名 → 本地路径的映射，是 GitHub App 集成的数据基础
- `skillUsage` / `toolUsage`：行为统计，Anthropic 用来了解哪些功能被真正使用
- `appleTerminalBackupPath`：执行终端配置时会备份 Terminal.plist，Claude Code 做了这件事
- `hasIdeOnboardingBeenShown`：记录了 VSCode/PyCharm/IntelliJ 的 IDE 插件引导是否已展示
- `s1mAccessCache`：某个高端模型（s1m 是内部代号）的访问权限缓存

---

## 与 settings.json 的分工

官方文档的分工是明确的：

| 文件 | 适合存放 |
|---|---|
| `~/.claude/settings.json` | 用户级**主动配置**（权限、环境变量、模型偏好等） |
| `.claude/settings.json` | 项目级共享配置（可以提交到 git） |
| `.mcp.json` | 项目级 MCP 服务器配置（可以提交到 git） |
| `~/.claude.json` | 运行时状态（程序自动维护，**不建议手动改**） |

---

## 关键洞察

1. **"不可见的功能"比"可见的配置"更多**。用户能看到和控制的只是冰山一角，大量行为由服务端下发的 feature flag 决定，本地缓存只是快照。

2. **Claude Code 是一个重度追踪使用行为的产品**。从每个 session 的帧渲染 p99 到每个工具的使用次数，数据粒度相当细。这些数据用于产品决策，`isQualifiedForDataSharing: false` 可以选择退出部分数据共享。

3. **用 `numStartups` 和 `tipsHistory` 可以倒推产品的引导逻辑**。比如某个 tip 在第 287 次启动时出现，说明它的触发条件是"启动次数 >= 287"，而不是功能使用之后。

4. **projects 记录可以当作自己的 Claude Code 使用日志**。花费、token、行数变更等数据都在，可以写脚本从中提取使用统计。
