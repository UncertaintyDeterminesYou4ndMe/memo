# Claude Code 请求断连抓包诊断指南

## 背景

使用 Claude Code 时，如果遇到请求被「上游断掉」的情况（如 `Connection reset by peer`、`TLS handshake failed`、`timeout` 等），需要抓包定位问题发生在哪一层。

典型场景：
- Claude Code 配置了自定义 API Endpoint（公司内网代理或中转服务）
- 请求在 TLS 握手后被重置
- 不确定是网络层、TLS 层还是应用层问题

---

## 核心问题分析

### 问题分层诊断

```
应用层 (HTTP)    ← Claude Code 发送 JSON 请求
      ↓
TLS 层          ← 证书验证、密钥交换
      ↓
TCP 层          ← 连接建立、数据传输
      ↓
网络层 (IP)      ← 路由、DNS 解析
```

当遇到 `Connection reset by peer`，通常是：
- **TLS 层问题**：证书无效、TLS 版本不匹配
- **TCP 层问题**：防火墙、代理主动断开
- **应用层问题**：代理转发失败、认证失败

---

## 方案一：tcpdump 抓包（快速定位网络层）

### 1. 基础用法

```bash
# 抓取与特定主机的所有流量
sudo tcpdump -i <网卡名> host <TARGET_HOST> -w /tmp/capture.pcap

# 实时查看（不保存文件）
sudo tcpdump -i <网卡名> host <TARGET_HOST> -n -tttt
```

### 2. 常用过滤条件

```bash
# 只抓 443 端口（HTTPS）
sudo tcpdump -i en0 'port 443' -w https_traffic.pcap

# 抓取特定 IP 和端口
sudo tcpdump -i en0 'host <IP地址> and port 443' -w filtered.pcap

# 抓取 TCP flags（看 SYN/ACK/RST）
sudo tcpdump -i en0 'tcp[tcpflags] & tcp-rst != 0'  # 只显示 RST 包
```

### 3. 分析抓包文件

```bash
# 读取 pcap 文件
tcpdump -r /tmp/capture.pcap -nn

# 统计连接状态
tcpdump -r /tmp/capture.pcap -nn 'tcp[tcpflags] & tcp-syn != 0' | wc -l
tcpdump -r /tmp/capture.pcap -nn 'tcp[tcpflags] & tcp-rst != 0' | wc -l
```

### 4. macOS 网卡选择

```bash
# 查看网卡列表
ifconfig | grep -E "^(en|lo)"

# 常用网卡
# en0: Wi-Fi（最常用）
# en1-en6: 有线网卡或 Thunderbolt
# lo0: 本地回环（测试本地代理时用）
```

---

## 方案二：curl 快速诊断（无需抓包工具）

### 1. 测试直连连通性

```bash
# 详细输出连接过程
curl -v --connect-timeout 10 https://<TARGET_HOST>/v1/models 2>&1

# 只看 TLS 握手和连接时间
curl -w "\nDNS: %{time_namelookup}\nConnect: %{time_connect}\nTLS: %{time_appconnect}\nTotal: %{time_total}\n" \
  -o /dev/null -s https://<TARGET_HOST>/v1/health
```

### 2. 分层诊断检查清单

```bash
# Step 1: DNS 解析
dig <TARGET_HOST> +short

# Step 2: ICMP 连通性（确认网络层）
ping -c 3 <TARGET_HOST>

# Step 3: TCP 端口连通性（确认传输层）
nc -zv <TARGET_HOST> 443

# Step 4: TLS 握手详情
openssl s_client -connect <TARGET_HOST>:443 -servername <TARGET_HOST>

# Step 5: 完整 HTTP 请求
curl -v -H "Authorization: Bearer <YOUR_API_KEY>" \
  https://<TARGET_HOST>/v1/models 2>&1 | head -50
```

### 3. 诊断输出解读

| 输出 | 含义 | 问题层级 |
|------|------|----------|
| `Trying <IP>...` → 无响应 | DNS 解析成功但 TCP 连不上 | 网络/防火墙 |
| `Connected` → `SSL handshake` 卡住 | TLS 协商失败 | TLS 证书/版本 |
| `SSL handshake` 成功 → `Connection reset` | 代理/上游服务拒绝 | 应用层 |
| `HTTP 403/401` | 认证失败 | API Key/权限 |
| `HTTP 502/503/504` | 代理/网关错误 | 上游服务问题 |

---

## 方案三：mitmproxy 中间人代理（查看明文请求）

### 1. 安装与启动

```bash
# macOS
brew install mitmproxy

# 启动代理（HTTP/HTTPS）
mitmproxy --mode regular --listen-port 8080

# 或后台模式
mitmweb --mode regular --listen-port 8080 --web-port 8081
```

### 2. 配置 Claude Code 走代理

```bash
# 设置系统代理环境变量
export HTTPS_PROXY=http://localhost:8080
export HTTP_PROXY=http://localhost:8080

# 如果使用自签名证书，跳过验证（仅测试）
export CLAUDE_SSL_VERIFY=false

# 启动 Claude Code
claude
```

### 3. 分析请求

在 mitmproxy 界面中：
- 查看请求 URL、Headers、Body
- 查看响应状态码和返回内容
- 搜索特定关键词（如 `/v1/messages`）

---

## 方案四：Wireshark 图形化分析

### 适用场景

- 需要可视化查看 TLS 握手流程
- 分析 TCP 重传、乱序等问题
- 导出特定流进行分析

### 基本流程

1. **抓包**：选择网卡 → 开始捕获 → 复现问题 → 停止
2. **过滤**：使用 `host <IP>` 或 `tls` 过滤
3. **追踪**：右键流 → Follow → TCP Stream / TLS Stream

### 常用过滤表达式

```
# 过滤特定主机
host <IP地址>

# 过滤 TLS 握手
ssl.handshake.type == 1  # Client Hello
ssl.handshake.type == 2  # Server Hello

# 过滤 RST 包
tcp.flags.reset == 1

# 过滤完整会话
http.request.method == "POST" or tls
```

---

## 典型问题诊断案例

### 案例 1：TLS 握手后被 Reset

**现象**：
```
* Connected to <TARGET_HOST> (<IP>) port 443
* SSL connection using TLSv1.3
* Recv failure: Connection reset by peer
```

**分析**：
- TCP 连接成功 → 网络层正常
- TLS 握手成功 → 证书正常
- 握手后立即 Reset → 代理/防火墙拦截

**可能原因**：
1. 代理需要额外认证（mTLS、Client Certificate）
2. SNI 被检测并拦截
3. 代理检测到非浏览器流量特征

**解决方案**：
```bash
# 检查是否需要 Client Certificate
openssl s_client -connect <TARGET_HOST>:443 -cert client.crt -key client.key

# 使用系统证书存储
curl --cacert /path/to/ca-bundle.crt https://<TARGET_HOST>
```

### 案例 2：DNS 解析正确但连接超时

**现象**：
```
* Trying <IP>:443...
* connect to <IP> port 443 failed: Operation timed out
```

**分析**：
- DNS 解析成功 → 域名配置正确
- TCP 连接超时 → 网络不可达或防火墙拦截

**排查步骤**：
```bash
# 1. 确认路由可达
traceroute <TARGET_HOST>

# 2. 检查本地防火墙
sudo iptables -L -n | grep 443  # Linux
sudo pfctl -sr | grep 443       # macOS

# 3. 测试不同出口
curl --interface <网卡名> https://<TARGET_HOST>
```

### 案例 3：代理返回 502/503

**现象**：
```
< HTTP/2 502
< server: nginx
{"error": "Bad Gateway"}
```

**分析**：
- 能连接到代理 → 代理存活
- 代理返回 502 → 代理无法连接上游服务

**排查**：
```bash
# 检查代理配置
curl -v --proxy-header "X-Debug: true" https://<TARGET_HOST>

# 尝试绕过代理直接测试上游（如果有权限）
unset HTTPS_PROXY
curl https://api.anthropic.com/v1/models  # 测试官方 API
```

---

## 环境变量与配置检查

### Claude Code 相关环境变量

```bash
# 查看当前配置
env | grep -iE "(anthropic|claude|proxy)"

# 关键变量
ANTHROPIC_BASE_URL=<YOUR_BASE_URL>      # 自定义 API 端点
ANTHROPIC_API_KEY=<YOUR_API_KEY>        # API 密钥
HTTPS_PROXY=<PROXY_URL>                 # 系统代理
HTTP_PROXY=<PROXY_URL>
NO_PROXY=<EXCLUDE_HOSTS>                # 不走代理的地址
```

### 快速测试配置

```bash
# 测试 1：官方 API（绕过所有代理）
unset ANTHROPIC_BASE_URL HTTPS_PROXY
curl https://api.anthropic.com/v1/models \
  -H "Authorization: Bearer <YOUR_OFFICIAL_API_KEY>"

# 测试 2：自定义端点（直连）
export ANTHROPIC_BASE_URL=<YOUR_BASE_URL>
unset HTTPS_PROXY
curl <YOUR_BASE_URL>/v1/models \
  -H "Authorization: Bearer <YOUR_API_KEY>"

# 测试 3：通过代理
curl -x <PROXY_URL> <YOUR_BASE_URL>/v1/models \
  -H "Authorization: Bearer <YOUR_API_KEY>"
```

---

## 调试工具速查表

| 场景 | 工具 | 关键命令 |
|------|------|----------|
| 快速连通性测试 | curl | `curl -v --connect-timeout 10 <URL>` |
| DNS 问题 | dig | `dig <HOST> +short` |
| TCP 端口测试 | nc | `nc -zv <HOST> <PORT>` |
| TLS 证书检查 | openssl | `openssl s_client -connect <HOST>:443` |
| 网络层抓包 | tcpdump | `tcpdump -i en0 host <IP> -w capture.pcap` |
| 明文 HTTP 分析 | mitmproxy | `mitmproxy --mode regular -p 8080` |
| 可视化分析 | Wireshark | 过滤 `host <IP>` |
| 路由追踪 | traceroute | `traceroute <HOST>` |

---

## 关键结论

1. **分层诊断**：从 DNS → Ping → TCP → TLS → HTTP 逐层确认问题所在
2. **先curl后抓包**：用 curl 快速定位问题，必要时再用 tcpdump/Wireshark 深入
3. **区分代理与上游**：502 是代理问题，RST 可能是代理策略，timeout 是网络问题
4. **环境变量隔离**：测试时逐个 unset 环境变量，确认是哪个配置导致的问题
