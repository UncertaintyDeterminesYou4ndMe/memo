# Claude Skills 多 Agent 同步方案

## 背景

Claude Code 的技能（Skills）存放在 `~/.claude/skills/`，但多个 Agent 共享使用时，每个 Agent 各有一份独立的技能目录，容易产生版本漂移：

- `~/.claude/skills/` —— 当前 Agent 的技能
- `~/.agents/skills/` —— 团队共享的技能池

两边各自独立更新，久而久之就会出现：某 Agent 用的是旧版 `frontend-design`，而另一边早已更新；或某 Agent 新增了一个技能，其他 Agent 完全不知道。

---

## 本次同步过程（手动 merge）

### 1. 对比差异

```bash
# 只在 ~/.agents/skills/ 有的（同事自定义技能）
comm -23 <(ls ~/.agents/skills/ | sort) <(ls ~/.claude/skills/ | sort)

# 只在 ~/.claude/skills/ 有的（官方技能）
comm -13 <(ls ~/.agents/skills/ | sort) <(ls ~/.claude/skills/ | sort)
```

发现：
- `~/.agents/skills/` 有 `code-cleanup`、`taste-skill`、`video-remove-vocals`、`xiaohongshu` 等自定义技能
- `~/.claude/skills/` 有官方 Anthropic 技能：`skill-creator`、`docx`、`pdf`、`pptx`、`xlsx`、`claude-api` 等，对方没有

### 2. 从官方仓库更新 `~/.claude/skills/`

官方技能仓库：`~/code/skills/`（`github.com/anthropics/skills`）

```bash
cp -r ~/code/skills/skills/* ~/.claude/skills/
```

### 3. 双向 merge

```bash
# 把我的官方技能同步给同事
for skill in algorithmic-art brand-guidelines canvas-design claude-api doc-coauthoring \
  docx frontend-design internal-comms mcp-builder memo-practice pdf pptx \
  remotion-best-practices skill-creator slack-gif-creator theme-factory \
  web-artifacts-builder webapp-testing xlsx; do
  cp -r ~/.claude/skills/$skill ~/.agents/skills/
done

# 从同事那里学习自定义技能
for skill in code-cleanup taste-skill video-remove-vocals xiaohongshu; do
  cp -r ~/.agents/skills/$skill ~/.claude/skills/
done
```

### 4. 处理有差异的共有技能

`ui-ux-pro-max` 两边都有但路径不同：

- `~/.agents/skills/` 版本：脚本用绝对路径 `~/.agents/skills/ui-ux-pro-max/scripts/search.py`（正确）
- `~/.claude/skills/` 版本：脚本用相对路径 `skills/ui-ux-pro-max/scripts/search.py`（错误）

保留 `~/.agents/skills/` 的版本，不覆盖。

---

## 长期同步方案：符号链接

手动 merge 只解决一次，治标不治本。**根本解法是让两个目录指向同一份数据**。

### 操作步骤

```bash
# 1. 备份（谨慎起见）
cp -r ~/.claude/skills ~/.claude/skills.bak

# 2. 删除 ~/.claude/skills 目录
rm -rf ~/.claude/skills

# 3. 创建符号链接，指向共享技能池
ln -s ~/.agents/skills ~/.claude/skills
```

### 验证

```bash
ls -la ~/.claude/skills
# 应显示：~/.claude/skills -> /Users/<username>/.agents/skills
```

### 效果

| 操作 | 结果 |
|------|------|
| 修改 `~/.claude/skills/xxx/SKILL.md` | 自动同步到 `~/.agents/skills/xxx/SKILL.md` |
| 同事在 `~/.agents/skills/` 新增技能 | 当前 Agent 立即可用 |
| 从官方仓库更新技能 | 一条命令，所有人受益 |

### 后续更新官方技能

官方仓库有更新时，只需：

```bash
cd ~/code/skills && git pull
cp -r ~/code/skills/skills/* ~/.agents/skills/
```

所有 Agent 自动同步。

---

## 技能描述优化要点

在本次同步过程中，顺手用 `skill-creator` 优化了 `code-cleanup` 的 description：

**原则**：Claude 倾向于「不触发」技能，description 要足够「有推动力」：

1. **覆盖多语言触发词**：中文场景下要同时包含中英文关键词
2. **描述隐式场景**：不只是关键词匹配，要描述"即使用户没说 cleanup，但明显需要"的情形
3. **`## 何时使用` 不要放在 body 里**：触发条件全部放在 frontmatter 的 description 字段

**改动示例**：

```diff
- description: 执行迭代期代码清理，识别并删除无用代码、重复逻辑和过期实现，保持小步安全变更。
+ description: 迭代期代码清理专家，识别并安全删除无用代码、重复逻辑、废弃实现和遗留兼容层，保持小步可验证变更。
+ Use this skill whenever the user mentions cleanup, dead code, unused imports, remove unused,
+ refactor, 清理, 删无用代码, 重复逻辑, or wants to tidy up code before a commit or code review.
+ Also trigger when the user finishes a feature and mentions the codebase has leftover old
+ implementations, legacy adapters, or deprecated paths — even if they don't explicitly say "cleanup".
```
