---
month: 202609
week: 4
date: 2026-09-24
type: 论文解读
slug: WhatWorkedBench
---

# 选对配置 ≠ 懂干预：实验理解要交整张效应表

你有没有过这种体验：research agent 跑完预算，交回一个「最优配置」和一段「我学到了什么」——榜上绿了，但你问「把开关 A 打开、在 B=off 时分数会怎样」它却答不上来，甚至自己写的规则和交的表互相打架？CMU / 清华这篇 WhatWorkedBench（arXiv:2609.27490）把 **实验理解** 定义成：在预算测量之后，对 **条件分量效应** 的预测准不准——赢配置 ≠ 懂干预。看完我的感受是：这和 [LLM-as-Judge](../week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) / [RuVerBench](./20260922_RuVerBench_论文解读_裁判验长轨迹Rubric仍有噪声.md) / [LayerIsolatedEval](./20260922_LayerIsolatedEval_论文解读_总分几乎不动切片却崩了.md) 同一条闭环主题——**分数 alone 不证明 discovery**；要用效应级理解 + 把程序结构塞进推理环。

## 核心摘要

WhatWorkedBench 评的不是「有没有刷到最高分配置」，而是 episode 结束时交付的完整响应面 \(\hat{u}(z)\)（所有二元掩码 \(z\in\{0,1\}^d\)，\(d=4\) 或 \(6\)）相对穷尽原生 CPU 参考的 **条件分量效应** 准不准。免费锚点是全 0 与全 1；其余最多再买 \(B\) 次测量。目录：**36** 个任务条件、**30** 数据源、**8** 类工作流（分类 / 回归 / 聚类 / 预测 / 复原 / 检索 / 心拍检测 / 链路预测），**1248** 条配置记录、**3392** 个条件效应；**35/36** 条件存在符号翻转交互。主指标效应 MAE \(E_t\) 与恢复 \(R_t=\max(0,1-E_t/A_t)\)。关键结果：pair ridge 在 \(B=8\) 时 **15/22** 选到精确最优，但只有 **3** 个通过严格重建（每个效应误差 ≤ 分数全距的 10%）——**choice ≠ understanding**。同一批 Flash agent 观测上换 Shared GP，恢复 **0.632→0.698**（original）与 **0.621→0.720**（additional）。编码代码等价（inactive 参数）把六因子 GP 恢复在 \(B=20\) 从 **0.248→0.462**。Agent 能口头报出规则，交的表仍违反；事后投影 **0.338→0.507**。

## 论文信息

- **标题**：WhatWorkedBench: Benchmarking Experimental Understanding in AI Agents
- **作者**：Jingjie Ning、Xueqi Li、Yibo Kong（CMU）、Dongting Li（Tsinghua；通讯相关标注见原文）
- **机构**：Carnegie Mellon University · 清华大学
- **链接**：https://arxiv.org/abs/2609.27490
- **观察时间**：2026-09-24（HF Daily Papers）

---

## 🎯 为什么这件事值得写

科研 agent 越来越会：选型、跑实验、改 recipe、把便宜试探迁到昂贵配置。可靠实验要求的不只是「找到一个好点」，而是 **预测干预在新背景下如何表现、如何与其它改变组合**。单点最优或平均主效应掩盖交互：图 2 里检索开关 binary counts 在 TF off 时抬 NDCG@10 **+0.160**，TF on 时反而 **−0.084**，交互约 **−0.245**。目录里 **35/36** 条件有符号翻转；要求双向效应都超全距 5% 时仍有 **24** 条。

既有 bench（MLAgentBench、MLE-bench、ScienceAgentBench、FIRE-Bench 等）多评执行成功与结果物；BoxingGym / Gravity-Bench 等评实验设计与预测，但少有对 **穷尽原生干预参考下的完整条件效应场**。WhatWorkedBench 把交付物钉成一张完整预测表，用 CPU 穷尽当真值——对 Byron（judge / evals / research harness）：这是「发现认证」缺的那一维。

## 🏗️ 机制：响应面、条件效应、共享推断

任务 \(t\) 上原生执行定义确定性效用 \(u_t(z)\in[0,1]\)。Agent 可检查工作流代码与因子语义，自适应或批量购买 ≤\(B\) 个新配置；工具含查证据、买 mask、跑数值 Python、提交预测表（或点名公共 ridge helper）。权威观测写入已测格子后再算效应。

对分量 \(i\)、背景 \(z\)（\(z_i=0\)）：

\[
\Delta_i(z)=u_t(z+e_i)-u_t(z),\qquad
E_t=\frac{1}{d_t 2^{d_t-1}}\sum_i\sum_{z:z_i=0}\lvert\widehat{\Delta}_i(z)-\Delta_i(z)\rvert.
\]

\(A_t\) 为真效应绝对值均值；恢复 \(R_t=\max(0,1-E_t/A_t)\)。完整条件效应 + 一个锚点可重建整张表；路径一致性要求从 **同一张提交表** 导出所有预测效应。严格重建：每个条件效应误差 ≤ \(\max(10^{-12},\tau\cdot\text{range})\)，开发端 \(\tau=0.1\)。

协议还拆开三件可独立操纵的事：**测量选择**、**数值推断**、**程序结构**（如「过滤器关则宽度无效」的等价类）。Shared-estimator 固定同一批观测，只换 D1/D2 ridge 或 GP——用来回答「证据够不够」vs「用得够不够好」。

## 🧪 关键证据：选择成功 ≠ 效应理解

**数值对照（四因子，\(B=8\)）**：pair-effect ridge 家庭宏恢复约 **0.612**，**15/22** 精确选优，严格重建仅 **3**；其中 **13** 例「选对了但效应误差超严阈」。Effect-variance GP 恢复约 **0.701**、精确选优 16、严格 1——平均恢复与选优更强，最大误差准则上 pair ridge 更严。容差剖面（2.5%–40%）：pair ridge 通过 1/1/3/8/20 of 22。

**Agent（DeepSeek-v4 Flash/Pro，四因子）**

| Model | Cohort | \(B\) | Recovery | Exact | Strict |
| --- | --- | --- | --- | --- | --- |
| Flash | Original | 8 | **0.632** | 5 | 1 |
| Flash | Additional | 8 | **0.621** | 2 | 0 |
| Pro | Original | 8 | 0.366 | 3 | 0 |
| Pro | Additional | 8 | 0.503 | 6 | 1 |

Flash \(B=8\) 共 18 episode：7 精确最优、1 严格重建。未测格子用观测均值填，恢复仅 **0.287** vs 交付 **0.632**——重建不只是抄已测边。

**Shared GP 抬同一批观测**：original Flash **0.632→0.698**；additional **0.621→0.720**。说明测量策略与推断器要分开评：agent 采到的证据，换更好的估计器还能再榨一截。

**程序结构**：六因子 \(B=20\)，编码等价类把 pair-ridge **0.157→0.389**、effect-GP **0.248→0.462**；\(B=32\) 时 GP **0.537→0.697**。结构诊断八集：agent **口头报全规则**，交表却用 ridge helper 并 **违反等价**；事后按类投影 **0.338→0.507**，八跑 raw 效应误差全降——**认识规则 ≠ 把规则写进交付物**。

**扩展家族**：beat detection + link prediction 上六份完整 Flash 提交（\(B=32\)），Shared GP 把家庭宏恢复 **0.303→0.455**。

## 🔬 最有意思的部分

1. **构造性反例**：精确选优可与任意弱的效应恢复共存（附录两比特表）——协议从定义上就拆开两个目标。
2. **交互是常态不是边角**：35/36 符号翻转 → 只报「最好配置」或平均主效应会系统性撒谎。
3. **Shared inference 是评测设计贡献**：同一观测换估计器，才能说清 agent 输在采集还是输在重建。
4. **代码等价是免费信息**：inactive 参数把测量预算变成「代表元」——harness 该把程序结构当一等公民，而不是只塞进提示词。
5. **口头合规 vs 表合规**：规则能复述、表仍违规——和 RuVerBench「看起来满足」同一失败家族；需要 **可执行约束投影** 进提交闸门。

## 🤔 我的判断

定位：**实验理解的可执行评测基础设施**（效应场 + 选择 + 交付），兼自适应设计与程序结构推断平台。亮点三：

1. 把「懂不懂实验」从叙事改成完整条件效应误差；
2. choice vs understanding、acquisition vs inference 的拆解干净；
3. 代码等价与事后投影给出可抄的 harness 门禁。

局限：因子维数固定在 4/6；工作流是冻结原生库配方，不是开放研究创意；agent 侧主测 DeepSeek-v4 Flash/Pro，存在 rate-limit / 超时交付缺失；严格重建极少说明当前预算下「全场理解」仍极难。对 Byron（research agent / evals / 闭环）：**别用「刷到最优」当发现证书**——至少要：(a) 效应级或交互级切片；(b) 提交前强制程序等价投影；(c) 把 Shared GP / ridge 当默认重建层，而不是指望模型临场口算整张表。这与「Rubric/二值检查 → 代码检查 → Golden → 结构化 judge」的建造顺序同构：WhatWorkedBench 是 **discovery 层的代码检查**。

一句话收束：科研 agent 可以选对配置却完全不懂干预——**理解要交整张效应表，分数 alone 什么也证明不了**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
