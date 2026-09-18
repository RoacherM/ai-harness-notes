---
month: 202609
week: 3
date: 2026-09-18
type: 论文解读
slug: Meta-Harness
---

# 脚手架也能端到端搜：Meta-Harness 用完整轨迹文件系统替代压缩反馈

你有没有过这种体验：同一底座模型，换一套 harness（存什么、取什么、怎么塞进上下文），榜上能差出一个数量级——论文开篇就引用「固定 LLM 上 harness 可造成约 **6×** 性能差」。行业里 harness engineering 仍主要靠人手翻失败、改启发式；文本优化器（OPRO、TextGrad、AlphaEvolve、GEPA…）又把反馈压成标量、短摘要或滑动窗口。Stanford / MIT / KRAFTON 这篇 Meta-Harness（arXiv:2603.28052）问的是：**外环能不能直接搜 harness 代码，并让 proposer 通过文件系统读到所有历史候选的源码、分数与执行轨迹？** 看完我的感受是：它不是又一个 prompt 进化器，而是把「评测轨迹当一等公民」写进优化接口——和 Byron 关心的 eval harness / meta-harness 闭环几乎同构。

## 核心摘要

Meta-Harness 把 proposer 做成 coding agent：每次从含全部先验候选的 filesystem 里用 grep/cat 等按需读取，提出新 harness → 评测 → 把代码、推理轨迹与分数整目录落盘，循环。相对先前方法每步约 **0.002–0.026 MTok** 反馈，本文设置下单次评估诊断信息可达约 **10 MTok**（约三个数量级）。在线文本分类上，发现的 harness 相对 SOTA 上下文管理系统 ACE 提升 **7.7** 分，同时上下文 token 约 **4×** 更少（**11.4K** vs ACE **50.8K** / MCE **28.5K**），且用约 **4** 次评估就追上次优文本优化器跑 **60** 次的终局；检索增强数学推理上，单一发现 harness 在 **200** 道 IMO 级题、**5** 个 hold-out 模型上平均 **+4.7** 分；Agentic coding 的 TerminalBench-2 上，发现 harness 在 Claude Opus 4.6 达 **76.4%**（超手写 Terminus-KIRA **74.7%**），在 Haiku 4.5 达 **37.6%**（超 Goose **35.5%**，该模全部公开 harness 中 #1）。消融表明：scores-only / scores+summary 的 best 约 **41.3 / 38.7**，全量轨迹接口 median/best 到 **50.0 / 56.7**——**摘要补不回被压掉的诊断信号**。

## 论文信息

- **标题**：Meta-Harness: End-to-End Optimization of Model Harnesses
- **作者**：Yoonho Lee, Roshen Nair, Qizheng Zhang（Stanford）；Kangwook Lee（KRAFTON）；Omar Khattab（MIT）；Chelsea Finn（Stanford）
- **机构**：Stanford · MIT · KRAFTON
- **链接**：https://arxiv.org/abs/2603.28052 （2026 年 3 月 30 日提交）· 项目页 https://yoonholee.com/meta-harness/ · TerminalBench 产物 https://github.com/stanford-iris-lab/meta-harness-tbench2-artifact

---

## 🎯 为什么「文本优化器」配不上 harness 工程

Harness 的决策是长程的：现在存什么、何时检索、如何呈现，会在很多步之后才爆。压缩反馈删掉的，往往正是「把下游失败追到上游 harness 决策」所需的因果线索。Table 1 把对比写得很清楚：OPRO/TextGrad/AlphaEvolve/GEPA/Feedback Descent/TTT-Discover 多半是 Window / Last / Summary；Meta-Harness 是 **Full — all logs and scores**。

关键设计选择不是「用更强模型瞎变异」，而是：

1. **Proposer = coding agent**（而非裸 LLM），因为经验量迅速超过上下文；
2. **历史以 filesystem 暴露**，按需检索，而不是塞进一个巨型 prompt；
3. **外环刻意极简**——不绑死固定 scaffold / 发现档案格式，让 agent 自己决定查什么。

项目页上的小规模 TerminalBench 演示更直观：从 Terminus-KIRA **28.5%** 的难子集出发，约第 7 轮到 **46.5%**，proposer 对着原始日志做反事实诊断。正式全文设置里，最重任务上 proposer 每轮中位读 **82** 个文件、引用 **20+** 个先验候选。

## 🏗️ 搜索环：Propose → Evaluate → Store

图 2 三步：(1) agent 读 filesystem 提出 harness 代码；(2) 在评测任务上跑；(3) 提案代码、推理轨迹、分数写入新目录。搜索种群可从 zero-shot / few-shot / ACE / MCE / Terminus 等强基线初始化。信用分配发生在 **harness 层**（改外部代码），不是改权重——和 meta-learning / credit assignment 文献同一精神，但是 coding-agent 时代才跑得动的反馈带宽。

## 🧪 三项主实验

### 在线文本分类

LawBench（215 类）/ Symptom2Disease（22）/ USPTO-50k（180）；20 轮 × 2 候选 = 40 个 harness。相对手写 ACE：**48.6%** vs **40.9%**（**+7.7**），上下文 **11.4K** vs **50.8K**。相对 OpenEvolve / TTT-Discover：约 **0.1×** 评估次数追上，终局再高 **10+** 分。OOD 九数据集平均 **73.1%**，超 ACE **70.2%**（**+2.9**）。发现谱系包括 Draft Verification（低上下文）与 Label-Primed Query（高准确）等 Pareto 点。

### 检索增强数学推理

语料 ≥50 万已解题；搜索 40 轮、109 候选；用 GPT-OSS-20B 在搜索集选单一 harness，再转到 200 道未见 IMO 级题，并测 4 个搜索未见模型。相对 no-retriever 平均 **+4.7**；整体优于固定 BM25（约 **+1.3**），且避免 dense retrieval / random few-shot 在部分模型上的回退。关键：优化的是 **检索策略代码**，仍站在同一 BM25 栈上，而不是另训 dense encoder。

### TerminalBench-2（89 任务）

与公开竞赛同设定搜索（作者承认无 hold-out split，用人工/正则审计防题面泄漏）。Opus 4.6：**76.4%**（Terminus-KIRA **74.7%**，榜上约 #2，唯一更高的 ForgeCode **81.8%** 作者未能仅从公开仓复现）；Haiku 4.5：**37.6%** vs Goose **35.5%**（#1）。早期轨迹里 proposer 曾把结构改与 prompt 改混在一起导致双双回退，随后显式拆开混杂因素，转向更安全的加法修改——这是 filesystem 因果诊断的定性证据。

## 🔬 最有意思的部分：接口消融与「评测栈」隐喻

Table 3 消融几乎是整篇的灵魂：给代码+分数不够；再加 LLM summary 甚至可能更差（best **38.7** < scores-only **41.3**）；只有 **raw traces** 把 median 拉到 **50.0**。这对做 judge / eval harness 的人是直接警告：**把轨迹摘要进仪表盘，可能正在扔掉优化信号。**

与 X 上「Agent Eval / Agent Harness / Eval Harness / Meta-Harness」分层叙事对齐时，这篇给出可操作定义：Meta-Harness = **用完整经验优化 Agent Harness 的外环**；不可变检查、轨迹保全、hold-out（文本分类与数学里做了；TerminalBench 作为 discovery problem 另说）决定你信不信发现结果。

## 🤔 我的判断

定位：**自动化 harness 工程的方法论文 + 强工程样本**。亮点三：

1. 把反馈带宽从「摘要级」拉到「文件系统级」，并用消融证明轨迹不可省；
2. 跨分类 / 数学检索 / agentic coding 三域，数字扎实（7.7 / 4.7 / TB2 #1 Haiku）；
3. 外环极简，可迁移到「优化 skills、MCP 封装、上下文策略」等同构搜索。

局限与代价：搜索昂贵（每步可达千万 token 诊断）；TerminalBench 无严格 hold-out，存在基准特化风险；发现的 harness 可读性/可维护性要人工审计；proposer 本身依赖强 coding agent。对 Byron 的栈：若你在搭 judge 闭环或「harness 也要版本化/可搜索」，这篇应与 ComposeCL、LLM-as-Judge 并列——**ComposeCL 讲权重侧持续学习组合，Meta-Harness 讲脚手架侧用轨迹闭环自动改代码**。

一句话收束：下一个杠杆往往不在再微一次权重，而在**让优化器看见完整失败轨迹，并有权改掉决定上下文的那层代码**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
