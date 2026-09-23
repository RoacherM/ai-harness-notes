---
month: 202609
week: 4
date: 2026-09-23
type: 论文解读
slug: RRSI
---

# Harness RSI 会过拟合演化集；正则提案与筛选才保住 OOD，还更省 Token

你有没有过这种体验：用 LLM 一轮轮改 prompt / 控制流 / 工具描述 / 记忆，evolve 集分数往上飙，换个基准却回到原点甚至掉到初始脚手架 \(H_0\) 下面——像在背题而不是学机制？Google Cloud AI Research + UNC / Stanford / WashU 这篇 RRSI（arXiv:2609.24972）把问题说死：agent-system 级的 recursive self-improvement（RSI）若无约束地复用有限 evolve 反馈，会同时干三件坏事——**基准特化、追噪声、堆复杂度**。他们的解法不是缩小可编辑空间，而是**正则化搜索轨迹**：提案侧限预算与探索，筛选侧防泄漏、噪声与无谓涨成本。看完我的感受是：这正好把 [Meta-Harness](../week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 的「能搜」接到 [EvoHarnessBench](../week3/20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) 的「会忘 / 会非平稳」——RRSI 问的是 **怎么让 RSI 留下的是可迁移机制，而不是 evolve 集纪念品**。

## 核心摘要

RRSI（Regularized Recursive Self-Improvement of Agent Harnesses）在保持 harness 全可编辑的前提下，约束「怎么提候选、怎么留下」。提案侧：余弦退火的编辑基数预算 \(b_t\)（早期允许多机制协同、后期稀疏可归因）、全历史证据式归因、停滞时强制探索未试组件。筛选侧：泄漏审查（拒任务名/答案/基准特化逻辑）、噪声地板 \(\hat S(H')\ge S^\star-\delta\)、复杂度接受 \(\Delta C\le\beta_0+\beta_1\Delta S\)（Ridge 式）、以及按窗口删无贡献组件的结构剪枝（Lasso 式）。八基准三域（编码 / agentic workspace / 工程设计）：演化分裂上最高 **+14.1**（Gemini 3.5 Flash @ Terminal-Bench 2.1：64.6→78.7）；OOD 上 agentic 三件套相对 \(H_0\) **+3.5～+4.7**（JobBench 36.0→40.7 等），Frontier-Eng **+4.3** Medal（相对 **+24.3%**）；相对无正则演化，最终 harness 约 **30%** 更少 policy token（消融表：2.42M vs 3.80M /trial）。相对四家先验（Meta-Harness / AHE / TTHE / HarnessX），RRSI 往往 **evolve 增益最小，却是唯一 OOD 均值明显越过 \(H_0\)** 的（43.6 vs 39.7）；若干基线 OOD 甚至低于起点。

## 论文信息

- **标题**：RRSI: Regularized Recursive Self-Improvement of Agent Harnesses
- **作者**：Peng Xia*、Rujun Han、Zifeng Wang、Yanfei Chen、Yufan Zhang、Yoonho Lee、Chengsong Huang、Han Yu、Zhongying CuiZhu、Yifei Ming、Huaxiu Yao、Burak Gokturk、Tomas Pfister、Chen-Yu Lee（* Google 访问）
- **机构**：Google Cloud AI Research · UNC-Chapel Hill · Stanford · Washington University in St. Louis
- **链接**：https://arxiv.org/abs/2609.24972 · 代码 https://github.com/google-research/rrsi · 项目 https://regularized-rsi.com/
- **观察时间**：2026-09-23

---

## 🎯 为什么这件事值得写

现代 LLM agent 是系统：冻结骨干 + harness（提示、控制流、工具、记忆、上下文管理）。产品增益大量来自 harness 工程；自动化演进（Meta-Harness、AHE、TTHE、HarnessX 等）把「读失败轨迹 → 改脚手架」做成 RSI。但 Figure 1 的故事很残酷：evolve 增益大头留不住，若干方法 OOD 落到 \(H_0\) 以下。

和 Byron 侧已归档的 [RSI-Harness / Pi Genome](../week3/20260916_RSI-Harness_项目解读_切场景就是切Genome_Pi脚手架的可版本化.md)、[Meta-Harness](../week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 同一条河：差别在于 RRSI **不争论编辑什么，而争论搜索动力学该不该正则**——开放 \(\Omega(H)\)，约束轨迹。

## 🏗️ 机制：正则的是搜索，不是假设空间

记 \(S(H;\mathcal D)\) 为任务表现、\(C(H;\mathcal D)\) 为 policy-token 成本。标准环：在 Devolve 上跑 → 反馈 → 无约束提案 \(P_0\) → 按经验分选最优。问题是候选依赖**同一批任务的自适应复测**（Dwork et al. 式 holdout reuse）。

RRSI 两刀：

| 侧 | 手段 | 经典类比 |
| --- | --- | --- |
| 提案 | 退火编辑预算 \(b_t=b_{\min}+(b_{\max}-b_{\min})\cdot\frac12(1+\cos(\pi t/T))\) | \(L_0\) 基数 |
| 提案 | 全历史正负证据进 proposer | adaptive data analysis |
| 提案 | 停滞窗口内强制探未试组件 | 熵/多样性 |
| 筛选 | critic 先拒泄漏与惰性机械 | anti-memorization |
| 筛选 | \(\hat S\ge S^\star-\delta\) | 噪声地板 |
| 筛选 | \(\Delta C\le\beta_0+\beta_1\Delta S\) | Ridge / \(L_2\) |
| 筛选 | 无正贡献组件标删除 | Lasso / \(L_1\) |

实现上策略冻结 Claude Opus 4.8（编码域另跑 Gemini 3.5 Flash）；proposer / 失败分析 / 泄漏 critic 同为 Opus；基座 Terminus-2（编码）或 ReAct+MCP 工具带 + ReSum 式上下文（Harvey / EngDesign）。

## 🧪 主结果：evolve 克制，OOD 反而赢

相对 \(H_0\)（同窗测量）：Terminal-Bench **+6.0**，EngDesign **+4.9**，Harvey evolve **+1.1**；SWE-bench Verified（从未打分）**+1.8**；Harvey ID held-out **+2.3**；JobBench / GDPval / APEX **+3.5～+4.7**；Frontier-Eng **+4.3**。**没有任何 held-out 回退**——这正是「背题 harness」的典型失败模式。

Table 1（agentic workspace）刺眼：Meta-Harness evolve 最高（93.0），OOD 平均只 +0.9；AHE / TTHE OOD 掉到 \(H_0\) 以下；RRSI evolve 90.5 并不耀眼，OOD 均值 **43.6 vs \(H_0\) 39.7**。工程设计侧用冻结仿真器打分，排除「讨好 LLM judge」捷径——增益仍在。

消融（Table 2）：去接受侧 → evolve 91.5、OOD 41.0、token 3.59M；去提案侧 → OOD 41.9；两边都去（无正则）→ evolve 拉到 **92.8**（最高）但 OOD **40.3**≈\(H_0\)，token **3.80M** vs RRSI **2.42M**。

跨策略：Gemini 上 Terminal **+14.1**、SWE **+2.2**；把 Gemini 搜出的 harness 原样挂到从未参与搜索的 Gemini 3.1 Flash Lite，Terminal 11.2→14.6（相对 **+30.4%**）——机制不完全绑死在搜它的骨干上。

## 🔬 最有意思的部分

1. **「evolve 涨得少」可以是特性**：正则故意不吃噪声与泄漏带来的虚高；榜单若只报 evolve，会把 RRSI 判输。
2. **成本不是附赠**：无正则与先验方法都落在「更多 token、更低 OOD」被 RRSI 支配的区域；AHE 约 3.82M/trial，比 RRSI 多 **58%** token 却少 4.4 OOD 分。轨迹步数 RRSI **26.3** vs 先验 27.3–34.6（\(H_0\) 仍最省：1.56M / 21.2 步）。
3. 与 [SoL-Pi](../week3/20260918_SoL-Pi_论文解读_先省token再扩RSI_四个机制把Pi流量砍半.md) 同向：先管住 token / 复杂度，再谈 RSI 扩张——否则自我改进买的是测试时算力，不是机制。

## 🤔 我的判断

定位：**把经典正则化搬进 harness RSI 搜索动力学的方法论文**，测量故事与机制故事同样完整。亮点三：

1. 明确「过拟合是 harness RSI 的一阶病」，并用 OOD / ID held-out / 确定性仿真器三角验证；
2. 提案+筛选双侧正则，消融干净，且最终 harness 更轻；
3. 跨骨干、跨弱模型转移，说明留下的更像程序机制而非权重共谋。

局限：骨干冻结，不覆盖边训边进化；仍依赖有限 evolve 集与超参；更长地平线 / 异质工具生态还需外推。对 Byron（MMP Pi、关心 RSI / Meta-Harness / EvoHarnessBench）：若你在跑脚手架自改进，**默认加上泄漏 critic、噪声地板、成本接受曲线和剪枝**——否则 Meta-Harness 式完整轨迹搜索很容易变成 evolve 刷分器。和 EvoHarnessBench 拼：那边量「脚手架越加越忘」，这边给「怎么加才不容易忘」。

一句话收束：RSI 可以改一切，但 **不该让有限反馈改到一切**——正则提案与筛选，换来的是 OOD 与更少 token，不是更漂亮的 evolve 曲线。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
