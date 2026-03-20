# Clash 固定代理节点——防止 Anthropic 因 IP 乱跳封号

## 背景

使用 Claude Code 时，担心出口 IP 频繁变动导致 Anthropic 风控封号。
核心问题：Clash 的自动选择组（`url-test` / `fallback`）会定期切换节点，导致 IP 不固定。

---

## 核心判断：直连节点 vs 代理组

这是理解 Clash 路由的关键区别：

| 类型 | 配置 | IP 行为 |
|------|------|---------|
| 直连节点 | rule 直接指向 `proxy-name` | **固定**，不会跳 |
| url-test 组 | 每隔 N 秒测速，自动选最快 | **会跳**，IP 不固定 |
| fallback 组 | 首选节点挂了才切换 | 通常固定，挂了会跳 |
| select 组 | 手动选择，持久化 | **固定**，最佳平衡 |

**关键洞察**：`♻️自动选择` 这类 `url-test` 组是 IP 乱跳的根本原因。
把敏感服务（Anthropic、OpenAI）的规则指向 `select` 组或直连节点，才能固定出口。

---

## 现状发现

检查 longbridge.yaml 后发现，Anthropic 流量**已经**被正确处理：

```yaml
rules:
  - 'DOMAIN-SUFFIX,anthropic.com,<fixed-node-name>'
  - 'DOMAIN-SUFFIX,claude.ai,<fixed-node-name>'
  - 'DOMAIN-SUFFIX,claude.com,<fixed-node-name>'
  - 'DOMAIN-SUFFIX,claudeusercontent.com,<fixed-node-name>'
```

规则目标是**单个节点名**（非组），IP 不会自动切换。不需要改动。

---

## 如何正确配置（从零开始）

### 方案一：直接指向固定节点（简单，无法切换）

```yaml
rules:
  - 'DOMAIN-SUFFIX,anthropic.com,my-us-node'
```

缺点：节点挂了只能手动改配置。

### 方案二：select 组（推荐）

```yaml
proxy-groups:
  - name: Anthropic
    type: select          # 手动切换，不自动跳
    proxies:
      - us-node-01        # 默认选这个（面板展示第一个为当前选中）
      - us-node-02
      - jp-node-01

rules:
  - 'DOMAIN-SUFFIX,anthropic.com,Anthropic'
  - 'DOMAIN-SUFFIX,claude.ai,Anthropic'
  - 'DOMAIN-SUFFIX,claude.com,Anthropic'
  - 'DOMAIN-SUFFIX,claudeusercontent.com,Anthropic'
```

需要配合 `store-selected: true`（在配置顶层设置），Clash 重启后记住选择。

```yaml
profile:
  store-selected: true
```

---

## 跨配置使用节点（proxy-providers）

如果有两份配置，想在一份里用另一份的节点作为备用：

### 方式 A：复制节点（简单）

直接把另一份配置的 `proxies` 条目复制过来，重命名避免冲突：

```yaml
proxies:
  # 原有节点...

  # 从另一份配置复制的备用节点
  - { name: 'backup-us', type: trojan, server: <server>, port: 443, password: <pwd>, ... }
```

### 方式 B：proxy-providers（优雅，自动同步）

```yaml
proxy-providers:
  backup-nodes:
    type: file
    path: ./other-config.yaml   # 引用另一份配置的文件
    health-check:
      enable: true
      url: 'http://www.google.com/generate_204'
      interval: 300

proxy-groups:
  - name: Anthropic
    type: select
    use:
      - backup-nodes        # 直接引用 provider 里的所有节点
    proxies:
      - my-fixed-node       # 也可以混入本地节点
```

优点：另一份配置节点更新时自动同步，不需要手动维护两份 proxies 列表。

---

## 规则匹配优先级

Clash 规则**从上到下**，第一条匹配即生效。
精确规则放前面，兜底规则放最后。

```yaml
rules:
  - 'DOMAIN-SUFFIX,anthropic.com,Anthropic'   # 精确规则，优先匹配
  - ...
  - 'GEOIP,CN,DIRECT'                          # 兜底
  - 'MATCH,DIRECT'
```

---

## 链式代理（relay）：出口 IP 永远固定

固定单节点有一个缺点：节点挂了只能手动换。
链式代理可以做到：前置节点自动切换（保证速度），出口节点固定（保证 IP 不变）。

```
请求 → 前置节点（url-test，自动选最快）→ 出口节点（固定 IP）→ 目标
```

Clash 配置：

```yaml
proxy-groups:
  # 前置：自动选最快
  - name: 前置节点
    type: url-test
    proxies: [node-a, node-b, node-c]
    interval: 60

  # 链式：前置 + 固定出口
  - name: Anthropic
    type: relay
    proxies:
      - 前置节点        # 第一跳，自动切换
      - fixed-exit      # 出口，IP 永不变

rules:
  - 'DOMAIN-SUFFIX,anthropic.com,Anthropic'
```

**实际判断**：如果现有固定节点稳定，不需要引入 relay 的额外复杂度。等节点出问题再考虑。

---

## Anthropic 封号的真正风险分层

根据实践总结，封号风险从高到低：

| 风险 | 说明 | 解决方案 |
|------|------|---------|
| **共享 IP 被污染** | 同一出口 IP 上有其他用户违规，整个 IP 被标记 | 换独享 IP 或住宅 IP 节点 |
| **IP 频繁变动** | 每次请求出口不同，触发风控 | 固定出口节点（已解决）|
| **高风险地区 IP** | 部分数据中心 IP 段被批量标记 | 用住宅 IP 或质量好的专线 |

**关键洞察**：固定 IP 只解决了"变动"问题，但如果节点是共享的数据中心 IP，其他人的滥用行为仍可能连累自己。真正安全的做法是用**独享 IP 或住宅 IP** 作为出口。

---

## 关键知识点总结

- **IP 乱跳的根因**：规则指向了 `url-test`/`fallback` 类型的代理组
- **最稳方案**：`select` 组 + `store-selected: true`，手动选定后持久化
- **敏感服务**：Anthropic、OpenAI 这类对 IP 一致性敏感的服务，不应走自动选择组
- **`proxy-providers`**：优雅地在多个配置间共享节点，避免维护重复的 proxies 列表
- **relay 组**：前置自动切换 + 出口固定，兼顾速度和 IP 稳定性
- **共享 IP 风险**：固定节点不等于安全节点，节点质量（是否独享）同样重要
