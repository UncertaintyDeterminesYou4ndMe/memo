# 定位 Dock 里反复跳动的 Electron 幽灵进程

## 背景

macOS Dock 里出现一个裸的 "Electron" 图标，反复出现、消失、又出现，像是有个后台应用一直在"跳"。第一反应容易以为是某个卡死的残留进程，但直觉不可靠——需要一条系统性的定位链来回答三个问题：**它是谁？谁拉起的？是卡死还是在正常干活？**

## 定位链：三步回答三个问题

### 第一步：先看完整命令行，不要先杀进程

```bash
ps aux | grep -i electron | grep -v grep
```

关键判断：**进程的完整命令行参数是最丰富的"自我说明"**，杀掉之前先读它。这一条命令直接给出了四条线索：

1. **可执行文件路径**：`.../<worktree>/node_modules/electron/dist/Electron.app/...` —— 不是系统里装的某个 app，而是某个 git worktree 里 `node_modules` 下的开发版 Electron；
2. **`--user-data-dir=/var/folders/.../T/<app>-e2e-XXXXXX`**：临时目录、带 `e2e` 字样 —— 这是测试环境，不是日常使用的实例；
3. **`-r .../playwright-core/lib/server/electron/loader.js`**：主进程是被 Playwright 的 loader 注入启动的 —— 拉起者是 e2e 测试框架；
4. **启动时间**：几分钟前 —— 是新近活动，不是几天前的僵尸。

到这里"它是谁"已经清楚：某个 worktree 里的 Playwright e2e 测试在启动真实的 Electron 应用。

### 第二步：两次采样看 PID，区分"卡死"与"活跃"

单次快照无法区分挂死的残留和正在推进的任务。隔一小段时间再查一次：

- 第一次采样：Electron 主进程 PID 为 A；
- 第二次采样：PID A 已消失，出现了新的 PID B，`--user-data-dir` 的随机后缀也换了。

关键判断：**PID 在轮换 = 生命周期正常推进**。Playwright 的每个 e2e 用例都会启动一个全新的 Electron 实例、测完退出、下一个用例再启动新的。这正是 Dock 图标"跳"的机制——不是一个 app 在请求注意（bounce），而是图标随实例反复出现→消失→再出现。

### 第三步：顺藤摸瓜找发起者

进程列表里除了 Electron 本体，还能看到整条调用链：

```
/bin/zsh -c source ~/.claude/shell-snapshots/snapshot-zsh-....sh ... && eval '...'
  └─ npx playwright test --config e2e/playwright.config.ts --workers=1
       └─ playwright worker
            └─ Electron（每个用例一个实例）
```

`shell-snapshots` 路径暴露了发起者是**另一个 Claude Code 会话**——它在 `.claude/worktrees/` 下的隔离 worktree 里跑性能验证。再读那条 zsh 命令的完整内容，甚至能还原它的意图：先跑一遍当前改动的 e2e 套件（AFTER），再 checkout 回 main 的 e2e 目录跑基线（BASELINE），做前后性能对比。

## 结论与处置

- **不是故障**：是一场合法的、正在推进的 e2e 基准测试，跑完自然安静。处置选项只有"等它跑完"或"确认该验证不再需要后杀掉 `playwright test` 进程树"。
- **治本方向**：e2e 拉起的 Electron 不该出现在 Dock。可在测试环境变量下调用 `app.dock.hide()`（或打包时设 `LSUIElement`），让测试实例对用户不可见。

## 关键知识点

1. **神秘后台进程的通用定位链**：`ps` 全参数（它是谁）→ `--user-data-dir` / 可执行路径归属（哪个环境）→ 进程树上溯（谁拉起的）→ 多次采样看 PID 轮换（卡死还是活跃）。每一步都只靠只读命令，定位完成前不做任何破坏性操作。
2. **命令行参数即身份证**：Electron/Chromium 系进程的参数里藏着大量上下文（user-data-dir、loader、remote-debugging-port、app-path），比进程名有信息量得多。
3. **Dock 图标"跳" ≠ 应用请求注意**：短生命周期进程高频重启，宏观上也表现为图标跳动。先分辨是 bounce 还是"重生"。
4. **Claude Code 并行会话的可观测性**：`.claude/worktrees/` 路径和 `~/.claude/shell-snapshots/` 是识别"这是哪个 agent 会话干的"的两个指纹，排查多会话互相干扰时非常好用。
5. **Playwright + Electron 的 e2e 形态**：Playwright 通过 `-r .../electron/loader.js` 以 require 注入方式启动 Electron 主进程，每个测试用例独立实例、独立临时 profile——资源开销和 GUI 副作用（Dock、焦点）都按用例数放大。
