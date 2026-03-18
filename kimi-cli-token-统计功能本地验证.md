# kimi-cli Token 统计功能本地验证

## 背景

PR #1485 为 kimi-cli 引入了 Token 使用统计功能（status bar 显示 today/week 累计），但 Codex Review 连续指出了多轮并发安全和数据完整性问题。需要在本地验证修复方案，同时确保用户历史数据不丢失。

---

## 核心问题：并发场景下的数据一致性

### 问题 1：并发写入覆盖（last-write-wins）

多个 kimi 会话同时运行时，各自的内存统计相互独立，写入磁盘时会互相覆盖，导致"all sessions"统计被低估。

**关键判断**：不在写入时做复杂合并逻辑，而是**每次读取前重新加载磁盘数据**。这样简单且可靠：

```python
def record(self, usage: TokenUsage) -> None:
    # 先重载，再添加，再保存
    self._reload_and_merge()
    self._daily.add(usage)
    self._weekly.add(usage)
    self._save()
```

### 问题 2：属性返回过时数据

`daily`/`weekly` 属性最初只返回内存缓存，如果其他进程已更新统计，当前进程仍显示旧值。

**关键判断**：属性访问时也触发重载，确保"all sessions"始终反映最新磁盘状态：

```python
@property
def daily(self) -> TokenUsage:
    self._reload_and_merge()  # 关键：读取前刷新
    return self._daily.to_token_usage()
```

### 问题 3：数据类型安全

JSON 文件用户可编辑，字段可能是字符串、浮点、甚至 `inf`/`nan`。

**关键判断**：不直接 `int()` 转换，而是用 `math.isfinite()` 守卫：

```python
def _to_int(v: Any) -> int:
    if isinstance(v, float):
        if not math.isfinite(v):  # 拒绝 inf/nan
            return 0
        return int(v)
    if isinstance(v, str):
        fv = float(v)
        if not math.isfinite(fv):
            return 0
        return int(fv)
```

### 问题 4：JSON 根类型验证

文件内容可能是合法 JSON 但非对象（如 `[]`、字符串、数字）。

**修复**：读取后先验证 `isinstance(raw_data, dict)`，否则视为空数据。

---

## 本地验证流程

### 数据安全优先

用户关心历史记录和 token 统计是否保留。验证发现：
- 用户数据在 `~/.kimi/` 下（user-history、token-stats.json、config.toml）
- `uv tool uninstall/install` 不影响该目录
- 仍执行备份：`cp -r ~/.kimi/user-history ~/kimi-backup-$(date +%Y%m%d)/`

### 版本切换

原本地安装是旧版本（1.22.0），需要升级到 1.23.0（PR 版本）：

```bash
# 从本地源码安装（可编辑模式）
uv tool install -e .

# 验证
kimi --version  # 1.23.0
```

### 并发测试验证

```python
# 模拟两个会话
ledger1 = TokenLedger(f)
ledger1.record(usage(100, 50))

ledger2 = TokenLedger(f)  # 新进程视角
ledger2.record(usage(200, 100))

# 会话1再次写入，应看到会话2的数据
ledger1.record(usage(50, 25))

# 验证：总计应为 350/175，而非 150/75
assert ledger1.daily.input_other == 350
```

---

## 核心洞察

1. **"all sessions" 的正确性依赖于读取时刷新**，而非写入时合并。后者复杂且容易遗漏场景（如属性访问）。

2. **用户数据与工具安装分离**是 CLI 设计的最佳实践。`~/.kimi/` 作为数据目录，升级工具时天然保留。

3. **Codex Review 的价值**：自动化审查能发现人类容易忽略的边界情况（并发、类型、异常）。四轮迭代后的代码鲁棒性远高于初版。

---

## 结论

- ✅ 修复了 4 轮 Codex Review 提出的问题
- ✅ 本地验证通过（单会话、并发会话、边界情况）
- ✅ 用户数据完整保留（历史记录、token 统计、配置）
- ✅ 推送到 PR #1485，等待合并

最终状态栏效果：
```
[kimi-for-coding · thinking]  ████░░░░░░ 36% (95.9k/262.1k) │ in:2.9m out:6.1k │ 18m
  today   2.9k     in:2.9k out:6.2k
  week    2.9k     in:2.9k out:6.2k
```
