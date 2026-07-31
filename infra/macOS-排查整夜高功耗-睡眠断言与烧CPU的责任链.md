# macOS 排查整夜高功耗：睡眠断言与烧 CPU 的责任链

## 背景

电脑挂机一晚，早上发现机身很烫、功率很高，想知道"是谁在用电"。

事后来看，这类问题的正确提问方式不是"哪个进程 CPU 高"，而是两个独立的问题：

1. **谁不让电脑睡觉**（没有这一条，后面的一切都不会发生）
2. **醒着的这一夜，谁在烧 CPU**

## 关键判断

### 瞬时采样不可靠，要用累计账

`top -l 1` 的第一次采样所有进程 CPU 都显示 0（top 需要两次采样做差才能算出 CPU%），单看瞬时值也无法证明"整晚"。真正有说服力的是 `ps` 的累计 CPU 时间：

```bash
ps aux -r | head -15
ps -eo pid,etime,time,pcpu,comm -r | head -20
```

用 `TIME / ELAPSED` 算平均占用：某浏览器渲染进程 16 小时累计 1167 分钟 CPU 时间，平均 ≈ 1.2 个核心一刻不停跑了一整夜——这本身就是"整晚记录"，不需要事先装任何监控。

### 耗电是责任链，不是单一进程

最终拼出的完整链条：

- **一个行情类客户端的音频会话没释放**，`coreaudiod` 替它持有 `PreventUserIdleSystemSleep` 断言长达 20+ 小时 → 电脑整夜无法入睡（这是前提）
- **浏览器某个标签页的渲染进程死循环**，平均 1.2 核烧了一整夜（这是主要热源）
- **系统趁机器"空闲且醒着"跑机会性任务**：`mediaanalysisd`（照片分析）和 `mds_stores`（Spotlight 索引）在整晚能耗榜上甚至排在浏览器前面（这是放大器）

三者只治任何一个都不彻底：关掉卡死标签页但机器仍不睡，系统任务照样跑；让机器能睡，则后两者自然消失。**根因是睡眠断言**。

## 三个证据源（按可信度排列）

### 1. pmset：谁在阻止睡眠，且能溯源到调用方

```bash
pmset -g assertions        # 当前断言
pmset -g log | grep -iE "PreventUserIdleSystemSleep|PreventSystemSleep" | tail -20   # 历史
```

关键细节：断言条目里有 `Created for PID: xxx`——`coreaudiod` 持有的音频断言会指回真正占用扬声器的应用 PID，不会把锅留在 coreaudiod 头上。断言的持有时长（如 `20:33:53`）直接就是"没睡多久"的证据。

### 2. ps 累计 CPU 时间：谁烧了多久

见上文。适合回答"这个进程是刚开始疯还是疯了一夜"。

### 3. powerlog 数据库：系统自带的逐进程能耗历史

macOS 一直在 `/var/db/powerlog/Library/BatteryLife/CurrentPowerlog.PLSQL` 记录逐进程能耗，**文件是 world-readable 的，不需要 sudo**，sqlite 只读打开即可：

```bash
DB="file:/var/db/powerlog/Library/BatteryLife/CurrentPowerlog.PLSQL?mode=ro"
sqlite3 "$DB" ".tables" | tr -s ' ' '\n' | grep -iE "energy|process|cpu"
```

有用的表：`PLProcessMonitorAgent_EventInterval_ProcessMonitorInterval`（主表只有时间戳）+ 同名 `_Dynamic` 表（ProcessName / PID / value），按 `FK_ID` join 后可以对任意时间窗口做逐进程汇总：

```sql
SELECT d.ProcessName, SUM(d.value) AS total
FROM PLProcessMonitorAgent_EventInterval_ProcessMonitorInterval_Dynamic d
JOIN PLProcessMonitorAgent_EventInterval_ProcessMonitorInterval p ON d.FK_ID = p.ID
WHERE p.timestamp BETWEEN <start_epoch> AND <end_epoch>
GROUP BY d.ProcessName ORDER BY total DESC LIMIT 15;
```

`value` 的绝对单位不明，但同窗口内的相对排名足够回答"昨晚谁用电最多"。正是这张表暴露了 `mediaanalysisd` / `mds_stores` 这两个瞬时看不到的整晚大户。

## 关键知识点

- macOS 阻止睡眠的机制是 power assertion，`pmset -g assertions` 看当前、`pmset -g log` 看历史；音频类断言会标注真正的调用方 PID
- 浏览器单标签页死循环只崩渲染进程，`kill <renderer_pid>` 或浏览器内置任务管理器（Chromium 系 `Shift+Esc`）单独处理，不影响其他标签
- `mediaanalysisd` / `mds_stores` 是"机器醒着才会跑"的机会性任务，挂机耗电异常时它们经常是隐形大户，但它们是症状不是根因
- powerlog SQLite 是排查"过去发生了什么"的免费历史数据源，不用事先跑 `powermetrics`
- 排查时注意 `caffeinate` 断言可能来自自己的 CLI 工具（如 AI 编程助手的后台任务），属正常现象，别误伤
