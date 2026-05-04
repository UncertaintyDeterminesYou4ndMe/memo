# Claude Context Statusline：自适应主题配色

## 背景

statusline 用 Nord Aurora 配色，所有次要信息（token 计数、分隔符、进度条空白、时间戳等）共用一个 `dim` 变量控制颜色。原始实现用 ANSI dim 属性 `\033[2m`，在深色终端上呈现为灰色，视觉层级合理但辨识度不高。用户希望把这些灰色文字改得更醒目。

---

## 核心问题：单一颜色无法适应两种终端背景

第一轮尝试直接把 `dim` 改为荧光黄 `rgb(204,255,0)`——在深色背景下确实醒目了，但**太亮**，长时间使用刺眼。如果用户切到浅色终端，亮黄在白底上几乎不可读。

核心取舍：**不存在一个在所有背景上都好看的颜色**，必须根据终端主题动态切换。

---

## 方案：macOS 外观模式自动检测 + 双色板

### 检测方式

```bash
if defaults read -g AppleInterfaceStyle >/dev/null 2>&1; then
    _dark=true   # 深色模式：该 key 存在且值为 "Dark"
else
    _dark=false   # 浅色模式：该 key 不存在，命令返回非零
fi
```

**关键判断**：选 `defaults read` 而不是 OSC 11 终端查询或 `COLORFGBG` 环境变量，原因：
- OSC 11 需要终端支持且有延迟，在管道场景（statusline 通过 stdin 接收 JSON）不可靠
- `COLORFGBG` 很多终端不设置
- `defaults read` 在 macOS 上最可靠，且 statusline 本身就是 macOS 场景

### 双色板设计

| 模式 | 主色调 | dim 色 | 设计思路 |
|------|--------|--------|----------|
| 深色 | Nord Aurora 原色 | `rgb(160,160,130)` 柔和琥珀灰 | 比原版 dim 稍亮，带暖色调，不刺眼 |
| 浅色 | 全套加深 | `rgb(140,140,120)` 中性暖灰 | 在白底上保持可读性，不会太重 |

浅色模式不是简单调暗，而是整套颜色重新选择——蓝、绿、红等都换成饱和度更高、明度更低的版本，确保在白色背景上对比度足够。

### 自动生效机制

statusline 脚本每次被 Claude Code 调用时都会重新执行，`defaults read` 在每次执行时检测当前系统外观。切换 macOS 深色/浅色模式后，下次 statusline 刷新即自动切换配色，无需重启。

---

## 踩坑：改了项目文件但实际运行的是另一份

修改了 `~/code/claude-context/bin/statusline.sh`，但 Claude Code 的 `settings.json` 配置指向的是 `~/.claude/statusline.sh`——两份文件内容不同步。调试时一直没生效，最后通过 `grep status ~/.claude/settings.json` 发现了这个路径差异。

**教训**：改配置类脚本前，先确认实际运行路径，而不是假设项目目录就是生效目录。
