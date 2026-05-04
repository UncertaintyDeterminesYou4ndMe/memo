# 如何使用 Mac 作为服务器运行 Dify Workflow

本文档介绍如何在 macOS 系统上配置定时任务，让 Mac 作为服务器持续运行 Dify workflow 或其他 Python 脚本。

## 前置准备

### 1. 准备 Python 环境

```bash
# 创建项目目录
mkdir -p ~/your-project
cd ~/your-project

# 创建虚拟环境
python3 -m venv venv

# 激活虚拟环境
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt
```

### 2. 准备 Python 脚本

确保你的 Python 脚本（如 `scheduled_runner.py`）能够正常运行：

```bash
python scheduled_runner.py
```

### 3. 创建日志目录

```bash
mkdir -p ~/your-project/logs
```

## 使用 launchd 配置开机自启动

macOS 使用 `launchd` 管理系统服务和定时任务，配置文件为 `.plist` 格式。

### 创建 LaunchAgent 配置文件

在 `~/Library/LaunchAgents/` 目录下创建配置文件：

```bash
nano ~/Library/LaunchAgents/com.yourapp.agent.plist
```

### 配置文件示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <!-- 服务标识符，必须唯一 -->
    <key>Label</key>
    <string>com.yourapp.agent</string>
    
    <!-- 要执行的程序和参数 -->
    <key>ProgramArguments</key>
    <array>
        <string>/Users/username/your-project/venv/bin/python</string>
        <string>/Users/username/your-project/scheduled_runner.py</string>
    </array>
    
    <!-- 工作目录 -->
    <key>WorkingDirectory</key>
    <string>/Users/username/your-project</string>
    
    <!-- 开机自动启动 -->
    <key>RunAtLoad</key>
    <true/>
    
    <!-- 进程异常退出后自动重启（保活） -->
    <key>KeepAlive</key>
    <true/>
    
    <!-- 标准输出日志 -->
    <key>StandardOutPath</key>
    <string>/Users/username/your-project/logs/stdout.log</string>
    
    <!-- 错误输出日志 -->
    <key>StandardErrorPath</key>
    <string>/Users/username/your-project/logs/stderr.log</string>
</dict>
</plist>
```

### 配置说明

| 配置项 | 说明 |
|--------|------|
| `Label` | 服务唯一标识符，通常使用反向域名格式 |
| `ProgramArguments` | 要执行的命令，第一个元素是程序路径，后续是参数 |
| `WorkingDirectory` | 脚本运行时的工作目录 |
| `RunAtLoad` | 设为 `true` 时开机自动启动 |
| `KeepAlive` | 设为 `true` 时进程退出后自动重启 |
| `StandardOutPath` | 标准输出重定向到的文件 |
| `StandardErrorPath` | 错误输出重定向到的文件 |

### 其他常用配置

```xml
<!-- 定时执行（每小时运行一次） -->
<key>StartInterval</key>
<integer>3600</integer>

<!-- 或者指定具体时间执行 -->
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>9</integer>
    <key>Minute</key>
    <integer>0</integer>
</dict>

<!-- 环境变量 -->
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/usr/local/bin:/usr/bin:/bin</string>
</dict>
```

## 加载和管理服务

### 加载服务（启动）

```bash
launchctl load ~/Library/LaunchAgents/com.yourapp.agent.plist
```

### 卸载服务（停止）

```bash
launchctl unload ~/Library/LaunchAgents/com.yourapp.agent.plist
```

### 查看运行状态

```bash
# 查看所有运行的服务
launchctl list

# 查看特定服务
launchctl list | grep yourapp
```

### 查看日志

```bash
# 查看标准输出
tail -f ~/your-project/logs/stdout.log

# 查看错误日志
tail -f ~/your-project/logs/stderr.log
```

## 完全禁用开机自启动

如果要彻底禁用服务：

```bash
# 1. 停止并卸载服务
launchctl unload ~/Library/LaunchAgents/com.yourapp.agent.plist

# 2. 删除配置文件
rm ~/Library/LaunchAgents/com.yourapp.agent.plist

# 3. 验证是否已移除
launchctl list | grep yourapp
```

## 查看系统中的定时任务

### 查看用户级任务

```bash
# 查看 crontab
crontab -l

# 查看 LaunchAgents
ls -la ~/Library/LaunchAgents/

# 查看当前加载的服务（排除系统服务）
launchctl list | grep -v "com.apple"
```

### 查看系统级任务（需要 sudo）

```bash
# 查看系统级服务
sudo launchctl list | grep -v "com.apple"

# 查看系统级 LaunchDaemons
ls -la /Library/LaunchDaemons/
```

## 常见问题

### 1. 服务无法启动

- 检查 Python 路径是否正确：`which python`
- 检查脚本路径是否使用绝对路径
- 检查文件权限：`chmod +x your_script.py`
- 查看错误日志：`cat ~/your-project/logs/stderr.log`

### 2. 服务意外退出

- 检查 `KeepAlive` 是否设置为 `true`
- 检查 Python 脚本中是否有未捕获的异常
- 查看日志排查问题

### 3. 找不到模块或依赖

确保使用虚拟环境的 Python：
```bash
/path/to/venv/bin/python
```

而不是系统 Python：
```bash
/usr/bin/python3
```

### 4. 环境变量问题

在 plist 中显式设置环境变量：
```xml
<key>EnvironmentVariables</key>
<dict>
    <key>PATH</key>
    <string>/usr/local/bin:/usr/bin:/bin</string>
    <key>PYTHONPATH</key>
    <string>/Users/username/your-project</string>
</dict>
```

## LaunchAgents vs LaunchDaemons

| 特性 | LaunchAgents | LaunchDaemons |
|------|--------------|---------------|
| 配置位置 | `~/Library/LaunchAgents/` | `/Library/LaunchDaemons/` |
| 运行权限 | 用户级别 | 系统级别 |
| 运行时机 | 用户登录后 | 系统启动时 |
| GUI 访问 | 可以 | 不可以 |
| 适用场景 | 个人任务、需要 GUI | 系统服务、后台任务 |

## 最佳实践

1. **使用绝对路径**：所有路径都使用绝对路径，避免相对路径问题
2. **配置日志**：始终设置 `StandardOutPath` 和 `StandardErrorPath`
3. **错误处理**：Python 脚本中添加完善的错误处理和日志记录
4. **测试先行**：先手动运行脚本确保正常，再配置 launchd
5. **定期检查**：定期查看日志文件，确保服务正常运行
6. **资源管理**：避免长期运行的脚本占用过多系统资源

## 示例：Dify Workflow 运行脚本

```python
#!/usr/bin/env python3
import time
import logging
from datetime import datetime

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def run_dify_workflow():
    """运行 Dify workflow 的主函数"""
    try:
        logging.info("开始执行 Dify workflow")
        # 这里添加你的 Dify workflow 调用代码
        # 例如：调用 API、处理数据等
        
        logging.info("Dify workflow 执行完成")
    except Exception as e:
        logging.error(f"执行出错: {e}", exc_info=True)

if __name__ == "__main__":
    logging.info("服务启动")
    
    while True:
        try:
            run_dify_workflow()
            # 每小时执行一次
            time.sleep(3600)
        except KeyboardInterrupt:
            logging.info("收到停止信号，服务退出")
            break
        except Exception as e:
            logging.error(f"未预期的错误: {e}", exc_info=True)
            # 出错后等待 60 秒再重试
            time.sleep(60)
```

## 参考资源

- [launchd.info](https://www.launchd.info/) - launchd 完整文档
- [Apple Developer Documentation](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)
- `man launchd.plist` - 查看 plist 格式手册
- `man launchctl` - 查看 launchctl 命令手册
