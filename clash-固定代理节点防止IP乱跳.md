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

## 关键知识点总结

- **IP 乱跳的根因**：规则指向了 `url-test`/`fallback` 类型的代理组
- **最稳方案**：`select` 组 + `store-selected: true`，手动选定后持久化
- **敏感服务**：Anthropic、OpenAI 这类对 IP 一致性敏感的服务，不应走自动选择组
- **`proxy-providers`**：优雅地在多个配置间共享节点，避免维护重复的 proxies 列表
