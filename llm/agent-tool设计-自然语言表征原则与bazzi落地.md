# Agent Tool 设计：自然语言表征原则与一次完整落地

从观察 Claude 输出 bash 命令的习惯出发，推导出一组 agent tool 设计准则，并在自己的八字 agent（bazzi 引擎 + Electron 外壳）上完整落地了一遍。

## 起点：两个"多余"的输出习惯，各有机制

Claude 生成的 bash 命令有两个显著特点，看似冗余，实则都有明确的机制支撑：

**1. 第一行注释是 self-prompting。** 注释不是给用户看的（shell 也会忽略它）。通过 autoregressive 机制，模型先生成这行注释，它立刻成为 context 的一部分，用自然语言锚定 intent，引导后续 token 的生成方向——与 CoT 的机制本质相同。

**2. `echo` + `&& echo "success" || echo "failed"` 是表征转换。** 模型的原生表征是自然语言。exit code、HTTP status 这些 tool 域表征，在训练数据中与"下一步修复动作"的关联，远弱于 `failed`、`not found` 这些自然语言表述。翻译成 "server failed to start" 之后，模型在参数空间里有大量可依赖的先验。本质是把**低条件概率密度的表征转换成高条件概率密度的表征**。

**再抽象一层：这是在用 context tokens 替代 hidden state。** 模型的"思考"要么发生在 forward pass 的 activations 里（每次重算、易丢失），要么发生在 context 里（持久、可被后续 attention 看到）。注释、description、自然语言 status，都是把本该发生在 activation 层的推理"物化"到 context 里。TODO list、scratchpad、CoT、ReAct 全是同一个 pattern 的不同包装。

## 五条设计准则

**1. 失败路径比成功路径更值得花设计预算。** 成功时模型下一步基本确定，output 写成 `"ok"` 还是一段话差别不大；失败时才是 token distribution 最平坦、最需要引导的时刻。检验标准：**把错误信息单独拿给一个没有任何上下文的人看，他能不能知道下一步该干什么。**

**2. 混合表征，不要二选一。** 最优形态是自然语言 summary 在前、结构化 payload 在后。Summary 负责引导决策（高条件概率密度），payload 负责给后续步骤提供精确引用（id、路径不能靠自然语言转述，会被改写出错）。只给自然语言，模型引用时要"猜"；只给 JSON，回到裸数据问题。

**3. Description/intent 输入字段按"下游决策空间大小"决定加不加，不是标配。** 收益集中在动作型+高风险的 tool（bash、edit、deploy）；纯查询 tool 下游分支少，锚点没东西可引导。且要求写 intent（"确认新组件该放哪个目录"）而非复述（"列出 src 下的文件"），这个差别要写进字段的 schema description，否则模型默认生成复述型。隐藏代价：description 写错会把后续生成锁死在错误 intent 上，比裸命令更难纠偏——与 CoT hallucination 同类。

**4. 返回长度是设计参数。** 状态和异常用自然语言，数据本体能截断就截断，且截断必须显式（"共 200 行，以下是前 20 行"）。沉默截断最坏：模型会当成"这就是全部"来推理。

**5. Output 端可以再往前一步：预生成候选动作。** 不只说"端口被占用"，而是 "端口 3000 被 nginx(pid 8432) 占用。可选：1) kill 8432 2) 改用 PORT=3001"。相当于把"工具→决策"之间的推理链外置到 tool output 里，直接拉高正确后续动作的概率。

以及一条元原则：**这些设计不要凭直觉定，拿真实失败 trace 验**——看模型走偏是否集中在"收到 tool output 之后的第一个决策"，是则改 output 表征，改完用同样 case 回放对比。Tool output 设计本质是 prompt engineering，不做 eval 都是玄学。

## 落地实录：八字 agent 的工具层改造

对象：无状态 Rust 计算引擎（`tool` 子命令输出纯 JSON）+ Electron agent 外壳（execFile 调二进制的 tool wrapper）。

### 发现 1：领域自然语言天然是高密度表征

引擎输出 `flow_day_interactions: ["冲月支卯", "合时支辰"]` 而不是 `interaction_type: 3`——命理领域词在训练语料里有千年积累，与"冲突/变动"的语义关联极强。**如果领域本身有成熟的自然语言术语体系，直接输出术语就是最优表征**，不需要额外翻译层。

### 发现 2：最大的坑在 wrapper 丢弃了引擎的好错误

引擎侧其实已经有不错的 stderr 中文错误，但 wrapper 里 `execFile` 失败时取的是 `error.message`——只有 `"Command failed: bazzi ..."`，引擎那句可行动的错误**根本没传给模型**。这是一类极易发生的架构性丢失：**每一层转手都可能把高密度表征降级成低密度表征，错误信息的"密度"要做端到端审计，不能只看产生端。**

修复后的契约：

- 引擎 tool 模式 stdout 永远是一个 JSON 对象——成功为数据，失败为 `{"error": "<中文可行动信息>"}` + exit 1。错误信息含非法输入原文 + 正确格式示例（"月份是否在 01-12、02-30 不存在……"），不透传 chrono 的英文原文。
- Wrapper 单流解析，提取链：stdout JSON error → stderr 文本（兼容旧二进制）→ execFile message（最后兜底）。

### 发现 3：摘要格式化的单一真相源应该在引擎，不在 harness

外壳原本在 TS 里手工拼"今日盘面摘要"注入 system prompt——领域知识（哪些字段重要、怎么压缩）被写进了 harness，换个消费方就得重写。改为引擎在 JSON 顶层输出确定性一行 `summary`（只聚合事实，不做吉凶判断，守住"数学归引擎、语义归模型"的分工），harness 直接取用。**混合表征的 summary 字段应该由离数据最近的一层生成。**

### 发现 4：准则 3 的反例应用

六个排盘工具全是无状态只读计算，一律**不加** description 输入参数——模型调用前的 intent 恒为"拿数据"。力气花在另一头：tool 的 schema description 写"何时用哪个"的路由知识（slim/full 的选择条件、"本轮已调过 chart 就用精简输出"、"禁止自行推算干支"）。**把调用策略前置到工具说明里，比让模型试错便宜。**

## 关键知识点

- 自然语言是模型参数空间中条件概率密度最高的表征形式，input 端和 output 端同理。
- Tool 设计的四个杠杆点，按 ROI 排序：错误信息的端到端密度审计 > output 混合表征（summary + payload）> schema description 里的路由知识 > input 端 intent 字段（仅动作型 tool）。
- 错误契约要写死到"流"级别：约定哪个流、什么格式承载错误，wrapper 才不会静默降级。
- 沉默截断、错误信息转手丢失、description 锁死错误 intent，是三种最隐蔽的表征降级。
