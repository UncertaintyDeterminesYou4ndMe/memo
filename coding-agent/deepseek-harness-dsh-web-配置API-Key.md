# DeepSeek Harness（dsh）Web UI 配置 API Key

## 背景

`npx @deepseek-ai/dsh web` 起 Web UI 后直接报没有 API key。DeepSeek Harness 是 DeepSeek 官方的 agent harness（口号 "Everything is a Plugin"），CLI 名叫 `dsh`，除了 `web` 还有 `chat` / `doctor` 等模式。

问题本身很小，但它的凭据分层设计值得记一下——和 Claude Code 那套「全靠环境变量」的路子不一样。

## 三种配置方式

### 1. Web UI 内配置（推荐）

启动后进 **Settings → Models** → 找到 DeepSeek 卡片 → 填 key → Save。

两个特性：

- **热生效**：改完下一个请求就走新路由，不用重启 server。
- **write-only**：key 存进去之后，页面拿回来的是脱敏描述符（redacted descriptor），永远读不到明文。

Key 从 DeepSeek 开放平台的 API keys 页创建（`sk-` 开头）。

### 2. 环境变量

```bash
export DEEPSEEK_API_KEY=sk-xxxxxxxx
npx @deepseek-ai/dsh web
```

### 3. 配置文件

两个文件，职责分离：

| 文件 | 作用 |
|------|------|
| `$DSH_HOME/.credentials.yaml` | 存密钥本体 |
| `$DSH_HOME/settings.yaml` | 只存**引用**，不存密钥 |

`settings.yaml` 里 provider 通过 `apiKeyEnv` 指向环境变量名：

```yaml
llm-pi-ai:
  providers:
    deepseek:
      apiKeyEnv: DEEPSEEK_API_KEY
```

`$DSH_HOME` 未设置时走默认目录。

## 关键设计：凭据与配置分离

这是这套东西真正值得记的一点：**settings 里永远只有 credential reference，密钥本体单独落在 `.credentials.yaml`**。

带来的直接好处是 `settings.yaml` 可以放心提交到仓库、跨机同步、贴到 issue 里——不用担心哪天手滑把 key 一起推上去。对比之下，很多 CLI 把 key 和模型配置混在同一个 JSON 里，结果就是这个文件整体变成敏感文件，没法共享。

`apiKeyEnv` 这个字段名也说明了取向：配置层描述的是「去哪儿取密钥」，不是「密钥是什么」。

## 排查线索

报错 `MISSING_CREDENTIAL` = 上面两条路都没走通：

1. Models 页面里没存过这个 provider 的 key，**且**
2. `apiKeyEnv` 指向的环境变量在进程环境里不存在

注意第 2 点的坑：`export` 只对当前 shell 生效。如果是 launchd / 系统服务拉起的 dsh，环境变量根本传不进去，这种场景下只能走方式 1 或直接写 `.credentials.yaml`。

## 参考

- 官方文档：`deepseek-harness.github.io/deepseek-harness/en/guide/providers`
- 仓库：`github.com/deepseek-ai/deepseek-harness`
