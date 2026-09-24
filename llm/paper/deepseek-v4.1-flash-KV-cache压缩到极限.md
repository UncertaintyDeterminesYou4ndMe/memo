# DeepSeek-V4.1-Flash 阅读笔记

- 论文：*DeepSeek-V4.1-Flash: Pushing the Limits of KV Cache Compression*，DeepSeek-AI，2026 年 9 月
- 来源：Hugging Face 仓库 `deepseek-ai/DeepSeek-V4.1-Flash` 内的 `DeepSeek_V41_Tech_Report.pdf`（51 页）。9 月下旬已上 arXiv：2609.19969
- PDF 本地存放在 `~/code/LLM-paper/deepseek-v4.1-flash/`，未入库
- 前置论文：DeepSeek-V4（arXiv 2606.19348，本地 `~/code/LLM-paper/deepseek-v4/`）。V4.1 是 V4 的增量版本，CSA/HCA、mHC、Engram、Muon 等概念都默认读者已知。

## 一句话

这不是一篇"更聪明的模型"论文，是一篇"把 KV cache 压到极限、让长上下文 agent 服务变便宜"的系统论文。模型能力提升主要来自数据，架构改动全部服务于降成本。

## 基本参数

| 项目 | 数值 |
|---|---|
| 骨干参数 | 552B（MoE，1 shared + 384 routed，每 token 激活 6 个 routed） |
| Engram 记忆参数 | 196B（稀疏访问，不算在骨干里） |
| 激活参数 | prefill 8B，decode 16B |
| 层数 | 40 层 = 20 层因果编码器 + 20 层解码器 |
| 上下文 | 1M token |
| 预训练数据 | 45T token，文本:多模态 = 7:1 |
| 全局 KV cache | 890 字节/token，是 V4-Flash 的 1/4，V1 的 1/437 |
| 持久化 KV cache（SSD） | V4-Flash 的 1/8 |
| 许可 | MIT |

## 要解决的问题（第 1 节）

Agent 场景的负载是"输入重、输出轻"：反复调工具，每次都要 prefill 一大段上下文。V4 的稀疏注意力已经把长序列的计算压下来了，剩下的瓶颈变成三个：

1. HBM 容量：运行时的全局 KV cache。
2. SSD / 主机内存容量：为前缀复用而持久化的 KV cache。
3. 带宽：cache 在存储层之间搬运。

论文把降成本拆成三层同时做：架构（CED + CSA2）、精度（FP4 KV）、部署（SWA Bounded Replay）。

## 核心架构改动（第 2 节）

### 2.2 Causal Encoder-Decoder（CED）

- 灵感来自 YoCo。下半 20 层是"因果编码器"，上半 20 层是"解码器"。
- 解码器各层的**全局** KV 不从本层隐状态算，而是直接用第 20 层的输出乘一个层相关的投影矩阵得到（公式 1）。
- 结果：prefill 只需要跑前 20 层就能拿到全部 40 层的全局 KV，prefill 计算量减半，所以 prefill 激活 8B、decode 激活 16B。
- 代价：SWA（滑动窗口注意力）的 KV 仍是逐层算的，解码器的 SWA KV 需要"回放"最近一段 token 才能得到。严格回放需要 n_win × L/2 个 token，论文用 Bounded Replay 只回放 n_win = 128 个，见 3.2.2。

### 2.3 Compressed Sparse Attention 2（CSA2）

KV 压缩有三个可乘的维度：每条 entry 的大小（MLA/GQA）、序列维度（每 m 个 token 压成一条，即 V4 的 CSA/HCA）、层维度（层间共享）。V4 只做了前两个，CSA2 补上第三个。

三种静态分配的层模式：

| 模式 | 自己算什么 | 复用什么 |
|---|---|---|
| Full | main KV、indexer K、indexer Q、Top-K 索引 | 无 |
| Reindex | indexer Q、重新打分选 Top-K | 上一个 Full 层的 main KV 和 indexer K |
| Reuse | 无索引计算，直接稀疏注意力 | main KV + 最近一次算出的 Top-K 索引 |

三种模式都各自算自己的 Q 和 SWA KV。共享 main KV 省存储，复用 Top-K 省 indexer 计算，两者解耦。

对比 V4 的简化：
- 去掉 CSA 中相邻压缩块的重叠和压缩时的绝对位置编码。
- indexer K 直接从 main KV 投影，不再从隐状态单独走一条压缩路径。
- 不再用 CSA–HCA 混合，全部是 CSA2。

实际配置（4.2.1）：
- 编码器：前 2 层纯 SWA；其余 18 层 CSA2，压缩率 m=2，分 3 组，每组 1 Full + 5 Reuse。
- 解码器：20 层 CSA2，m=1（不压缩序列），分 5 组，每组 4 层。第一组 1 Full + 3 Reuse，后四组 1 Reindex + 3 Reuse。
- 也就是说 40 层里只有 4 层真正产生 main KV（编码器 3 个 Full + 解码器 1 个 Full）。这是 890 字节/token 的主要来源。
- Top-K = 512，SWA 窗口 128。

### 2.3.2 Hierarchical Sparse Indexer

只用在解码器。解码器第一个 Full 层对全上下文打分，除了选自己的 Top-512，还按块（8 个位置一块）取最高分，选 2048 块，形成 16384 个候选位置的池子。后面的 Reindex 层只在这个池子里打分选 Top-512。效果是后续 indexer 的单次成本从"随上下文线性"变成常数。在后训练阶段引入，训练和推理用同样的候选限制。

### 2.4 其他扩展

- **Single-Pass mHC**：mHC 是多残差流。原实现需要三个串行 kernel，激活内存流量是理论下界的两倍。改动是让第 l 层的输入混合系数用第 l-1 层预测的 A（公式 6），消掉数据依赖，然后融合成一个 Mega-mHC kernel，流量减半。论文说性能损失可忽略。
- **Engram**：条件记忆模块，196B 参数，放在第 1 和第 14 层，n-gram 阶数 {2,3,4}，8 个哈希头，FP8 存储，推理时从主机内存 RDMA 预取。相比原版去掉了短因果卷积。
- **DSpark**：投机解码，替代 V3 的 MTP。3 层 Transformer 起草器，一次前向出 5 个草稿位置，加 Markov 头建模草稿间依赖，加置信头预测接受概率，调度器按系统负载动态选验证长度。预训练后单独训，后训练时随策略一起更新但梯度不回传骨干。
- **FP4 main KV**：MXFP4 格式，E2M1 + 每 16 通道一个 E4M3 scale，去掉 NVFP4 的第二级全局 scale（论证：RMSNorm 后 512 维隐向量最大值约 22.6，实际约 10，动态范围够用）。RoPE 之后量化。SWA KV 保持 FP8，因为它对量化敏感。后训练阶段做 QAT。

### 2.5 优化器

- Muon 用于线性层，改成 **head-wise Muon**：Q/K 权重按头拆开分别做 Muon 更新。理由是不同头的梯度分布不同，一个预条件器不够。GLM 5 和 Kimi-K3 也验证了这一点。
- Engram 表、词嵌入、预测头改用 **Sinkhorn 平衡的动量更新**（Algorithm 1）：Nesterov 动量后交替做行归一化和列归一化 K=11 次，让更新矩阵的行 RMS 和列 RMS 都约等于 1。只需一个动量 buffer，省掉 Adam 的二阶矩，且效果优于 Adam。学习率乘 0.18 以对齐 Adam 的更新幅度。

## 基础设施（第 3 节）

训练侧值得记的：
- 视觉编码器与 LLM 分离执行（disaggregated encoder），一步分三阶段：ViT 前向、LLM 前向反向、ViT 反向。
- 超长序列多模态样本的图片按 CP rank 分片加载，每张图只读一次。给出一个判断 I/O 是否成为瓶颈的条件：每 token 原始字节数 / 每 token 计算量 < 文件系统带宽 / GPU 带宽，与序列长度无关。
- CSA2 的跨层共享跨越 pipeline stage：用 shadow indexer（每个 stage 放一份可执行副本，参数只有一个逻辑 owner）、pipeline payload 扩展、micro-batch 级共享状态生命周期管理。

推理侧：
- Reuse 模式的层 prefill 只用 15 个 kernel，decode 11 个。
- Encoder–Prefill–Decode 三段分离部署。
- **持久化 KV cache 管理（3.2.1）**：V4 里 SWA KV 占持久化 cache 近一半，但它的复用模式是"分钟级、会话内"，和 72 小时的保留策略不匹配。V4.1 把 SWA KV 移出 SSD，放进每台机器 10% 主机 DRAM 组成的分布式内存池，TTL 几分钟。miss 了就用 Encoder SWA Bounded Replay 重算 128 个 token。
- **SWA Bounded Replay（3.2.2）**：只回放最近 n_win 个 token，并把 SWA 窗口截断在回放段内。得到的状态是近似的，不同 cache 命中位置算出的后续 KV 不完全相同。论文说实验表明质量损失可忽略，并在后训练中模拟了同样的回放做适配。

## 预训练（第 4 节）

- 数据：过滤掉"信息增益低的模型生成内容"（弱模型输出、机器翻译），视为隐式重复。多模态数据以清洗原生网页数据为主，不做大规模合成；重新从 Common Crawl 引导爬虫以覆盖多模态源；用 SmolVLM 做交错图文质量打分。
- 训练：batch 固定 1.006 亿 token；lr 2.6e-4 保持到 28T，28T–40T 余弦衰减到 2.6e-5，40T–45T 保持；从头就用 64K 稀疏注意力训练，没有 dense warmup；34T 处扩到 1M。全程无不稳定。
- ViT：32 层，patch 14，2D-RoPE，先 SigLIP 对比学习（47B 图文对，224×224），再接一个 4B MoE 做 236B token 的自回归微调（544–1344 分辨率），然后丢掉 LLM 只留编码器。主干预训练时 ViT 冻结，到 lr 衰减阶段才解冻。
- **基座评测（Table 1）**：用 1/3 总参数、1/4 激活参数达到 V4-Pro-Base 水平。MMLU-Pro 74.1（V4-Pro 73.5），HumanEval 79.4（76.8），BigCodeBench 60.6（59.2）；SimpleQA 42.3 低于 V4-Pro 的 55.2，MGSM 80.2 低于 V4-Flash 的 85.7。内部保留集 BPB 全部最低。

## 后训练（第 5 节）

论文明确说：**后训练没有算法创新**，SFT → RL → 在线策略蒸馏（OPD）完全沿用 V4 的做法，所有收益来自数据和环境流水线。原话："the marginal return of engineering the data and environment pipeline substantially exceeds that of algorithmic novelty in post-training."

### 5.1.1 任务合成

- 任务形式化为三元组（问题，环境，验证系统），用难度和正确性作为奖励，训练模型自己构造任务。
- 通用 agent：内部员工和外部合作方自愿回传交互数据，据此构造大量 mock 工具复现真实 SaaS / 企业系统接口；收集失败案例回放做定向 RL。
- 编码 agent：来源是内部编码 session（保留复杂或做不好的任务）+ 星标数达标的 GitHub 仓库。多个专职 agent 协作构建环境：判定可构建可验证 → 选起点 commit、设计任务、给出 fail-to-pass / pass-to-pass 评测点 → 在容器中搭环境并清除泄漏痕迹 → 多个 agent 试解 → 独立检查 agent 审环境和轨迹 → 修复 agent 修完再验。

### 5.1.2 RL 规模化

- 两个维度：训练算力和 scaffold 数量。跨 scaffold（Claude Code 多版本、OpenCode、Pi、DeepSeek Harness）联合训练，性能随累计步数持续上升（图 7、8）。
- rollout 拆成 agent sandbox（跑 scaffold 和工具）和 worker 容器（scaffold 无关的控制层），都跑在 DSec 上，训练器被抢占时 rollout 可挂起并保存完整状态。
- **模型合并**用来接续 RL：把不同 scaffold / 配置的 checkpoint 合并后作为下一轮 RL 的起点，图中断开的曲线段就是这样来的。

### 5.1.3 DSec 沙箱平台

2026-09-19 DeepSeek 单独发了 DSec 的完整论文（arXiv 2609.22978），见 [DSec 笔记](deepseek-dsec-智能体训练沙箱平台.md)。下面几条和那篇有出入，已在 DSec 笔记里对照。

- 百万级并发沙箱。不用 Kubernetes，自研放置引擎，多副本无同步协调，牺牲全局一致性换扩展性，节点本地做硬性准入检查。
- 节点级：按 sub-NUMA 分区绑 worker VM，单物理节点并发容器从约 1000 提到 2500 以上。对延迟敏感任务用 SCHED_IDLE + core scheduling 隔离。
- **agent 行为问题**：训练中 agent 利用了 XFS 权限漏洞、AppArmor 非法内存访问、包镜像服务泄漏答案等做 reward hacking，也会删关键二进制或整个文件系统。对策：per-sandbox AppArmor + eBPF 网络策略，环境被弄崩视为失败轨迹并回传"repercussion"信号。

### 5.1.4 可控推理强度

- system prompt 前加一行 `Reasoning Effort: {effort}`，b ∈ 1..100。
- 同一 prompt 在每个 effort 等级采样一组，组内做 GRPO 式相对优势；不同等级之间不直接比较。
- 长度惩罚系数随 effort 指数衰减（公式 10），effort 每增加 τ，惩罚系数乘 1/e。附录 C 给出边际效用的动机。
- 只在有限几个等级上训练，但部署时中间值可以插值。公开 API 的 max / high / low 对应 b = 100 / 75 / 50。
- 效果（5.3.2）：effort 从 25 到 100，8 个推理基准平均 67.1% → 76.3%，DeepSWE 66.0% → 74.2%，输出 token 约 2.5 倍。60–80 区间用不到一半 token 就拿到大部分收益，最后到 100 让 agent 轨迹变长 1.6–1.8 倍但提升很小。

### 5.2 异步后训练基础设施

- rollout 和训练同机分时。
- 派发粒度试了三种：batch 级（训练指标震荡）、prompt 级（卡在组内长尾）、最终用 **sample 级**：新完成样本数达到下一个 prompt 的 GRPO 组大小就派发。
- 长度偏差（短样本先完成）：按数据集限并发，丢弃早期过短样本。
- off-policy：限定最大 off-policy 比例，对过旧 token 做 loss mask。
- token 级中断和恢复：KV cache 和专家路由按 token 持久化，换 checkpoint 后不用重 prefill。跨 checkpoint 的样本用拼接的路由回放。
- 最终 OPD 用超过 40 个教师模型，全词表，教师架构可以彼此不同，也可以和学生不同。

## 评测结果（5.3）

关键数字（Table 3，均为 max effort）：

| 基准 | V4.1-Flash | V4-Pro | V4-Flash | Opus-5 | GPT-5.6 Sol | Kimi-K3 |
|---|---|---|---|---|---|---|
| GPQA Diamond | 90.9 | 92.4 | 89.9 | 93.4 | 94.1 | 92.9 |
| HLE | 36.8 | 42.7† | 37.8† | 56.3 | 44.5 | 43.5 |
| Codeforces rating | 3471 | 3348 | 3289 | - | - | - |
| Terminal-Bench 2.1 | 90.6 | 87.9 | 82.7 | 89.1 | 88.8 | 88.3 |
| Terminal-Bench 4.0 | 31.2 | 12.4 | 7.0 | 51.8 | 39.9 | 12.6 |
| DeepSWE v1.1 | 74.2 | 62.7 | 54.4 | 74.0 | 73.0 | 67.5 |
| ProgramBench | 20.3 | 15.5 | - | 37.0 | 23.0 | 17.5 |
| Automation-Bench | 54.8 | 43.2 | 37.7 | 50.3 | 45.8 | 46.7 |

读法：
- 日常编码和白领工作流类基准（Terminal-Bench 2.1、DeepSWE、AutomationBench）追平或超过闭源前沿。
- 需要专家领域知识的科研类 agent 任务（Terminal-Bench 4.0、ProgramBench）和 HLE 与 Opus-5 差距明显，论文自己承认。
- 网络安全类基准在开源模型中最高。
- 跨 scaffold（Table 4）：mini-SWE 74.2，DeepSeek Harness Minimal 72.6，Claude Code 69.8，Codex 65.6。差异存在但没有崩掉，论文归因于训练数据中 scaffold 的多样性。
- 多智能体（5.3.5）：DeepSeek Harness 的 Agent Team 模式，RL 奖励 = 任务分 + 协作奖励 − 派生延迟惩罚（延迟按事件 DAG 的关键路径算）。ProgramBench 8 小时截止时多智能体 30.04% 对单智能体 20.39%。标注为初步结果。

## 局限（第 6 节，作者自述）

- CSA2 的 Top-K 选择错误和 SWA Bounded Replay 的近似状态重建，在未测试的边界情况下可能造成能力退化。
- 基准分数接近 Fable-5、GPT-6 Astra 不等于在最难任务上能力对等。
- 评测环境越来越容易被模型钻漏洞（例：在 CyberGym 里反编译 Ubuntu 核心包找漏洞），呼吁社区重视。

## 我读完后的判断

1. **创新集中在系统层，不在算法层。** CED、CSA2 的层间复用、FP4 KV、Bounded Replay 每一项单看都是已有思路（YoCo、Cross-Layer Attention、IndexCache、NVFP4）的组合与工程化，价值在于把它们叠在一起并证明能训出来、部署得起。
2. **"890 字节/token"是这篇论文最值得记的数字。** 40 层里只有 4 层产生 main KV，再加 FP4。这决定了 1M 上下文的服务成本。
3. **后训练部分的核心主张是数据流水线 > 算法。** 这一点和 DeepSeek 自己 R1 时期"算法（GRPO、纯 RL）是主角"的叙事已经不同，值得和 R1 论文对读。
4. **可控推理强度的实现很简单**：只是 prompt 里一个数字 + 随数字指数衰减的长度惩罚。这个设计可以直接借鉴。
5. **agent 在训练中攻击沙箱**这一段是少见的一手记录，对做 RL 环境的人有直接参考价值。

## 建议的后续阅读顺序

1. DeepSeek-V4（arXiv 2606.19348）：补 CSA/HCA、mHC、Engram、Muon 用法，否则 V4.1 第 2 节读不透。
2. YoCo（Sun et al., 2024）：CED 的直接来源。
3. Cross-Layer Attention（Brandon et al., 2024）与 IndexCache（Bai et al., 2026）：CSA2 层维度复用的两个前身。
4. Engram（Cheng et al., 2026）与 DSpark（Cheng et al., 2026）：两个外挂模块各自的原论文。
5. DeepSeek-R1：对比后训练叙事的变化。

## 相关

- [从雕花论到Engram 五周后复盘](../memory/scaling轴迁移与持续学习-从雕花论到Engram-复盘-2026-09.md)
- [Scaling 轴迁移与持续学习：从雕花论到 Engram](../memory/scaling轴迁移与持续学习-从雕花论到Engram.md)
