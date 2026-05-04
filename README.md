# memo

个人技术备忘录：记录关键判断、设计取舍与核心洞察，而非操作流水账。

## 目录结构

| 目录 | 内容 |
|------|------|
| `claude-code/` | Claude Code 实现分析、调试与扩展（system prompt、skills、statusline、抓包诊断等） |
| `llm/` | LLM 训练与研究、prompt 工程（`prompts/`）、记忆机制笔记（`memory/`） |
| `data/` | 数据栈：ClickHouse、Kafka、ES、CDC、Flink、SQL 函数；按引擎细分 `paimon/`、`starrocks/`、`hbase/`、`duckdb/`、`postgres/` |
| `infra/` | 操作系统与语言基础：`linux.md`、`mac.md`、`java.md`、`python.md`、`go/`、`k8s/` |
| `tools/` | 工作流工具：git 快捷脚本、clash、dify、印刷色彩处理 |
| `archive/` | 历史归档：`memo.md` 旧大杂烩、临时财报文件等，新内容不再写入 |

## 写入约定

由 `memo-practice` skill 自动选择目录并写入。新主题若无对应分类，会在根创建新二级目录而非堆在根下。
