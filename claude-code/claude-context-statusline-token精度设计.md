# claude-context：Statusline Token 精度设计

## 背景

给 Claude Code 做一个状态栏插件，显示上下文窗口的实时 breakdown（系统提示词 / 工具 / 技能 / 消息 / 剩余空间）。核心挑战：Claude Code 不对外暴露每个分类的 token 数，只告诉你总量。所以得自己重建。

---

## 核心问题：外部重建 vs 真实 API 值之间的鸿沟

最初的设计是"外部黑盒估算"：

- 用 `cl100k_base`（OpenAI GPT-4 的 tokenizer）对每个来源分别计数
- 系统提示词、内置工具硬编码为 5,600 / 19,300 tokens
- 从 JSONL 会话日志里解析消息

这套方案的实际误差比预期大很多。在一个真实会话里，我们的估算是 38k，API 返回的真实值是 68k，**误差高达 56%**。

---

## 关键诊断：tool_result 放错地方了

排查发现：JSONL 解析代码把 `tool_result` 块放在 `assistant` 条目里处理，但实际上它在 **user** 条目里。

Claude API 的结构是：
- `assistant` 消息 → 包含 `tool_use`（工具调用请求）
- `user` 消息 → 包含 `tool_result`（工具执行结果，比如 bash 输出、文件内容）

bash 输出、文件读取这些大块内容全在 `tool_result` 里，全部漏掉了。修复后消息估算从 8k 跳到 31k，误差从 56% 降到 24%。

---

## 关键判断：不追求每分类精确，而是保证总量精确

Claude 的真实 tokenizer 没有公开的离线版本，唯一精确的方式是调 `anthropic.beta.messages.count_tokens()` API，但这要网络请求，不适合 status line。

**核心取舍**：状态栏已经从 Claude Code 那里拿到了真实的 API 总量（`input_tokens + cache_creation_input_tokens + cache_read_input_tokens`），而这个总量是 100% 准确的。

所以做了一个**比例校准**：
1. 用 tiktoken 估算各分类的相对大小（比例）
2. 把真实总量作为锚点，对所有分类等比缩放
3. 保证 breakdown 的 sum 永远等于真实 total

这样 total 精确，breakdown 的绝对值是估算但相对比例合理，视觉上有参考价值。

---

## Tokenizer 选择：cl100k_base 是合理的近似

`cl100k_base` 是 GPT-4 的 BPE tokenizer，不是 Claude 的。差异：
- 英文 / 代码：~5% 误差
- 中文 CJK：略大，`o200k_base`（GPT-4o）在中文上更接近，但两者对 Claude 来说都只是近似

没有更好的离线选择，但在有比例校准的前提下，tokenizer 选哪个对最终结果影响已经不大了。

---

## 核心洞察

1. **从外部重建上下文分布，最大的坑在数据结构**：不是 tokenizer 的锅，而是没有正确理解 Claude 的 API 消息格式（tool_result 归属）。读一遍 JSONL 里的真实结构比猜 API 文档更快。

2. **"总量精确 + 比例校准"比"每分类精确"更实用**：硬编码的常量值（sys prompt、tools）会随 Claude Code 版本变化，追求它们的精确是徒劳的。用真实 total 做锚点，问题的性质从"每个分类都要准"变成了"比例要合理"，后者容易得多。

3. **README 的诚实度很重要**：开源工具如果声称"精确计数"但实际有 56% 误差，会损害信任。明确写出哪些是精确的、哪些是估算的、误差大概是多少，反而更有价值。

---

## 最终效果

```
Opus 4.6 (1M context) ▊░░░░ 7% 67.8k/1.0m · claude-context (main)
  sys 7.4k · tools 25.4k · skills 6.6k · msg 42.2k · free 767k · buf 165k
```

- total（Line 1）：100% 精确，来自 Claude Code API
- breakdown（Line 2）：比例校准后 sum == total，各分类为估算值
- 源码：https://github.com/UncertaintyDeterminesYou4ndMe/claude-context
