# macOS 整夜休眠拖停 launchd 常驻服务

和《macOS 排查整夜高功耗：睡眠断言与烧 CPU 的责任链》正好是镜像问题：那篇是机器该睡不睡，这篇是机器不该睡却在睡。

## 背景

一个跑在 Mac 上的 launchd 常驻 daemon（多线程，各线程按自己的 cadence 轮询），最近夜里表现异常。用户的描述是：**"最近经常半夜网络自己断开"**。

结论先说：网络没断，是**整机在 Deep Idle 休眠**。凌晨 00:00–09:00 有 86% 的时间在睡，每 ~16 分钟只醒 2–20 秒。

## 关键判断

### 1. 用户描述里自带的归因，不能当前提

"网络断了"是一个**已经解释过的**现象，不是原始观测。原始观测只是"夜里任务没跑"。一旦接受"网络断"这个前提，接下来就会去查 Wi-Fi、路由器、DNS，全是死路。

正确做法是退回到可观测证据：任务**什么时候**该跑、**什么时候**实际跑了。

### 2. 日志没时间戳时，去找别的带时间戳的信源

daemon 的 stdout 有 26MB 却**一行时间戳都没有**，对"什么时候发生"这个问题完全无用。可用的替代信源，按可信度：

- **产物文件的 mtime**：同一类日报文件，以往都在 16:00 前后落盘，出问题那天落在了**次日 08:31** —— 直接量化出「迟到 20 小时」，一个 `ls -lat` 就够
- **业务库里的心跳表**：`process / last_beat / pid`，能看出哪些循环还活着
- **`pmset -g log`**：系统级、带时间戳、不依赖应用配合

教训反过来说就是：**常驻服务的 stdout 一定要带时间戳**，否则事后无法归因。

### 3. 心跳表的"按 cadence 分层失效"就是休眠的指纹

休眠不是把服务杀掉，而是把它的**时间轴拉长**。表现出来是：

| 循环周期 | 心跳表现 |
|---|---|
| 10–15s（对账、风控） | 看起来完全正常 |
| 1800s（半小时一次的分析） | 停摆 8 小时 |
| 600s（低频的合成任务） | 整整晚了 20 小时才跑 |

短周期任务在每次十几秒的清醒窗口里还能挤进去一轮，长周期任务几乎永远轮不到 —— `while True: work(); sleep(600)` 这种写法在 Deep Idle 下是被**饿死**的。

所以「快的循环没事、慢的循环停摆」这个组合，本身就该直接联想到休眠，而不是去查各个循环自己的代码。

### 4. macOS GUI 里根本没有"电池供电时不休眠"的开关

系统设置 → 电池 → 选项里那四项，和这个问题的关系：

| 面板项 | 对应 pmset | 有没有用 |
|---|---|---|
| 使用电源适配器供电且显示器关闭时，防止自动进入睡眠 | **AC** 侧的 `sleep` | 只管插电，**插电时本来就不睡** |
| 唤醒以供网络访问 | `womp` / `tcpkeepalive` | 没用，它产生的正是那些 2–20 秒 DarkWake，只够收推送，撑不住你的进程 |
| 使用电池时使显示器略暗一些 | `lessbright` | 无关 |
| 电池供电时优化视频流播放 | — | 无关 |

`pmset -g custom` 一看就明白：

```
Battery Power:  sleep 1     ← 空闲即睡
AC Power:       sleep 0     ← 永不睡
```

**电池那一侧的开关 Apple 已经从 GUI 拿掉了**，只能走 `sudo pmset -b sleep 0`（全局、费电）或 caffeinate（作用域小、不用 sudo）。

这也解释了为什么"插着电就没事"——不是网络环境不同，是走的 AC 那一侧配置。

## 踩坑：pmset log 按位置解析会把结论算反

这是本次最值得记的操作层教训。想统计"每天睡了多久"，很自然写成：

```bash
# ❌ 错的
pmset -g log | awk '$4=="Sleep" || $4=="Wake" || $4=="DarkWake" {print $1,$2,$4}'
```

问题在于日志里有两种以 `Wake` 开头的行，语义完全不同：

```
2026-XX-XX 04:03:57 +0800 DarkWake       DarkWake from Deep Idle ... 2 secs      ← 真实唤醒
2026-XX-XX 04:04:02 +0800 Wake Requests  [process=dasd request=... wakeAt=...]   ← 只是"预约了下次唤醒"
```

`Wake Requests` 是**计划**，不是**事件**。把它当成唤醒后，Sleep/Wake 配对整个反向，我第一版算出来是"每天清醒 37.64 小时" —— 一天没有 37 小时，这个物理上不可能的数才是发现 bug 的线索。

修正只要加一个字段判断：

```bash
# ✅ 对的
pmset -g log | awk '$4=="Sleep" || (($4=="Wake"||$4=="DarkWake") && $5!="Requests")'
```

还有一个容易搞反的语义细节 —— 行尾那个 `N secs`：

- `DarkWake ... 2 secs` → 这次**醒了**多久
- `Sleep ... 948 secs` → 这次**计划睡**多久

两者含义相反，认错方向结论也跟着反。

**通用教训**：解析半结构化日志时，(1) 同前缀不同语义的行必须用第二字段区分；(2) 统计结果出现物理上不可能的值，第一反应应该是回头查解析，而不是接受它。

## 方案：caffeinate 挂在常驻服务上的正确姿势

直觉写法是让 caffeinate 包住进程：

```bash
# ❌ caffeinate 当父进程
exec /usr/bin/caffeinate -i /path/to/venv/bin/python -m app.daemon
```

问题：**caffeinate 不转发信号**。launchd 停服务时 SIGTERM 打给 caffeinate，它自己死了，python 变孤儿继续跑 —— 下次启动会撞上单实例锁，或者更糟，两个实例并存。

正确写法是让 caffeinate 变成**兄弟进程**：

```bash
#!/bin/bash
set -a; source ~/path/to/app.env; set +a     # 顺带把凭据从 plist 挪进 env 文件

/usr/bin/caffeinate -i -w $$ &                # 绑定当前 shell 的 pid
exec /path/to/venv/bin/python -m app.daemon   # exec 后 python 继承同一个 pid
```

关键在 **`-w $$` 配合 `exec`**：`exec` 让 python 顶替 shell 占用同一个 pid，所以 caffeinate 用 `$$` 绑到的正是最终那个 python。于是：

```
launchd ─→ python (pid N)           ← 直接子进程，信号/KeepAlive 都正常
             └─ caffeinate -i -w N  ← 兄弟进程，目标退出后自动结束
```

`-w` 用的是 kqueue 的 `NOTE_EXIT`，被 SIGKILL 也能感知，不会留孤儿。

验证这个模式不需要动真服务：

```bash
bash -c '/usr/bin/caffeinate -i -w $$ & exec sleep 6' &
ps -eo pid,ppid,command | grep "caffeinate -i -w"   # 看 -w 的 pid 是不是 exec 后那个
# 6 秒后再看一次，应该已经自动消失
```

确认生效看断言，注意 `-w` 形式**不带 timeout**（`caffeinate -t N` 才有）：

```
pid 6254(caffeinate): PreventUserIdleSystemSleep
    Details: caffeinate asserting on behalf of Process ID 6249
```

**caffeinate -i 管不了合盖休眠** —— 它只挡 idle sleep。合盖还是会睡。

## 一个小手艺：改之前先记指纹

这次改动最后被要求回滚，需要把一段 1.6KB 的长凭据恢复回配置文件。两个习惯让回滚变得没有风险：

1. **不手抄**。凭据在另一处（env 文件）有单一来源，用脚本读出来重新生成配置文件，杜绝转录错误。
2. **改之前就记下哈希前缀**（`sha256[:12]`），回滚后逐项比对。

```bash
# 动手前先记一份，不打印明文
python3 -c "import hashlib;print(hashlib.sha256(open('x').read().encode()).hexdigest()[:12])"
```

配置文件里的凭据没法用 `git checkout` 恢复（这类文件通常 gitignore 或未跟踪），所以"事前指纹 + 事后比对"是唯一能证明**恢复得一字不差**的手段。顺带一提，`plutil -lint` 应该在每次改完 plist 后跑一次。

## 关键知识点

- `pmset -g custom` 看电池/AC 两侧的独立配置；很多"只在某些时候出现"的诡异现象，根子就是两侧配置不同
- Deep Idle 下机器每 ~16 分钟只醒 2–20 秒，`tcpkeepalive` 只维持 TCP 保活让系统能收推送，**不代表你的进程有 CPU 时间**
- launchd 的 `KeepAlive` 只在进程**退出**时重启它，进程被休眠冻住时它什么也不做 —— 别指望 KeepAlive 兜住这类故障
- `caffeinate -w <pid>` 比 `caffeinate <command>` 更适合托管常驻服务：不插进进程树、不吞信号、不留孤儿
- 常驻服务的 stdout **必须带时间戳**，否则出事后 26MB 日志一点用没有
- 排查这类"夜里不干活"的问题，先看心跳按 cadence 的分层失效模式，能一眼区分「服务挂了」和「服务被冻住了」
