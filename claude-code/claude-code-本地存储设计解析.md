# Claude Code 本地存储设计解析

## 背景

通过阅读 Milvus 博客文章（Claude Code 本地存储稳定性分析）并结合 `~/.claude.json` 实际数据，深入理解了 Claude Code 的状态管理设计。

---

## 核心设计：为什么选单文件 JSON 而非数据库？

零依赖原则 —— CLI 工具不能假设运行环境有 SQLite/LevelDB 等。单文件的好处：
- 用户可以直接 `cat ~/.claude.json` 查看/修改状态（unix 哲学）
- 写入前创建 `.bak` 备份，崩溃不丢配置

会话数据（对话历史）另用 **JSONL** 存储：每条消息独占一行，追加写入，不重写整文件。崩溃最多丢最后一条未完成消息。

---

## `.claude.json` 关键结构解析

### 用户身份

```json
"userID": "<sha256_hash>",
"firstStartTime": "2025-07-15T15:00:47.008Z",
"numStartups": 307
```

`userID` 是本地生成的哈希，不是 Anthropic 账号 —— 离线场景也能正常工作。

### Tips 频率控制（精巧的 UX）

```json
"tipsHistory": {
  "new-user-warmup": 1,
  "git-worktrees": 306
}
```

每个 tip 的值 = **上次展示时的 `numStartups`**。通过 `numStartups - lastSeen` 决定是否再次展示。`new-user-warmup: 1` 表示只在第1次显示，之后永不再现。

### projects —— 按绝对路径隔离状态

```json
"projects": {
  "~/code/my-project": {
    "allowedTools": [],
    "mcpServers": {},
    "lastCost": 0.11,
    "lastTotalCacheReadInputTokens": 39902,
    "lastTotalCacheCreationInputTokens": 19940,
    "lastSessionId": "<uuid>"
  }
}
```

以绝对路径为 key，每个项目权限完全隔离。`allowedTools: []` 表示未预授权任何工具。

**Token 缓存效率**：`cacheRead / (cacheRead + cacheCreation)` ≈ 67%，说明 prompt cache 命中率较高，节省了相当成本。

### Feature Flags 双轨：Statsig + GrowthBook

```json
"cachedStatsigGates": { ... },    // 旧系统，约 13 个 flags
"cachedGrowthBookFeatures": { ... } // 新系统，60+ 个 flags
```

**关键判断**：flags 被缓存在本地，离线时不会因无法请求实验平台而行为异常。`tengu_` 前缀是 Claude Code 的内部项目代号（天狗）。

两套系统并存 = 典型的"双写迁移"模式，Statsig 侧会逐渐废弃。

---

## 设计哲学总结

> 单文件 + 扁平结构 + 本地缓存 = 离线可用、崩溃可恢复、用户可理解

Claude Code 的"稳定感"不来自复杂的架构，而来自：
1. 会话数据 JSONL 追加写入（不重写全文件）
2. 配置变更前备份（最近 5 份 `.bak`）
3. Feature flags 本地缓存（不依赖网络）
4. 项目状态按路径严格隔离（权限不跨目录泄漏）
