---
month: 202610
week: 2
date: 2026-10-13
type: 论文解读
slug: LatentGRM
---

# 裁判别再逐字写评语：准则塞进潜空间，压九倍轨迹仍打平显式 SFT

你有没有过这种体验：线上要给一对回答做偏好判定，裁判模型先按正确性、指令遵从、清晰度逐条写评语，再汇总胜负；采样五票、九票，评测质量上去了，**解码成本也跟着翻倍**。你心里清楚：真正要的只是最后的偏好标签，中间那些准则级推理有没有必要一字一句吐出来？

苏州大学、剑桥、查尔姆斯、北大、清华与港科大这篇 **LatentGRM**（arXiv:2610.09788）给的答案是：把 rubric 引导的评价轨迹压进连续潜空间，在线只跑紧凑 latent 轨迹并直接输出 pairwise 偏好；需要可读评语时，再另起一个离线解释器从轨迹重建文本。看完我的感受：这是把 latent reasoning 第一次认真接到 generative reward model 上，而且用 rubric 干预证明「准则信息真的在 latent 序列里」，不是压完只剩黑箱分数。

## 核心摘要

LatentGRM 建立在语义分块、压缩与重建上。输入是用户请求、任务 rubric 与两个候选回答；显式裁判会生成整段文本评价再出胜负，LatentGRM 则把文本评价换成更短的连续状态序列 $z_1,\ldots,z_N$，再预测偏好。训练分两阶段：Stage 1 用 **Semantic Chunking** 按 rubric 块分配固定 latent 预算、再用语言边界细化切分，把教师评价轨迹蒸馏进词汇空间 latent；Stage 2 让裁判从评价输入自回归生成这些轨迹并输出偏好。单独的 **Latent Trace Interpreter（LTI）** 不在在线路径上，只从保存的轨迹离线重建评价文本。在匹配训练数据与 4B/8B backbone 下，vote@5 的 Avg.-8 上 LatentGRM-4B/8B 分别达到 **67.8 / 69.0**，对齐的显式 Rubric-RM 为 **67.0 / 68.1**；四个域上 LatentGRM-8B 把评价轨迹压到 **8.9–9.2×**，vote@5 总推理时间在 vLLM 下 **6.1–7.0×** 更快。准则干预实验里，改一条准则会使终点 log-odds 平均偏 **4.58**，把对应 latent 轨迹回放到基线 rubric 上仍能传到 **2.75**，说明偏好信息确实走在 latent 序列里。

## 论文信息

- **标题**：Judging in Latent Space: Efficient Generative Reward Modeling via Semantics-Preserving Compression
- **作者**：Mingqing Yuan, Xiaobo Liang, Junwei Yang, Ziwei Chen, Zeren Zhang, Hejin Wang, Yubin Wang, Juntao Li（通讯）
- **机构**：Soochow University · University of Cambridge · Chalmers University of Technology · Peking University · Tsinghua University · The Hong Kong University of Science and Technology
- **仓库**：https://github.com/EADMO/LatentGRM
- **链接**：https://arxiv.org/abs/2610.09788 （2026-10-08；HF Daily Papers 2026-10-12）

---

## 🎯 为什么这件事值得写

Generative Reward Model（GRM）越来越依赖「先写评语再下判」：多准则、多采样能抬准确率，但也把 autoregressive 解码成本堆到策略优化与响应筛选的热路径上。rubric-based RM 把评价拆成可点名的准则，结构更清晰，却往往更长。

| 路线 | 中间计算 | 痛点 |
| --- | --- | --- |
| 显式 GRM / Rubric-RM | token 级评语 + 证据 | 轨迹长、投票贵 |
| 标量 RM | 直接打分 | 难承载多准则推理与可解释反馈 |
| 通用 latent reasoning | 连续状态替代中间 token | 多在数学/通用推理上验证，少碰奖励建模 |

和 [JustAskJev](../../202609/week4/20260928_JustAskJev_论文解读_对齐失败检测别为每条准则再解码_一次问多题零样本就能筛.md) 想少解码、[TrustReviewer](../../202609/week3/20260921_TrustReviewer_论文解读_AI审AI审出来的裁判会塌缩_评分挤扁语义同质.md) 担心裁判塌缩同一条主线：**裁判贵且吵，就要压成本、保信息。** LatentGRM 的切口是：用 rubric 结构指导「哪里该压」，而不是盲目固定速率切块。

## 🏗️ 机制：语义分块蒸馏 + 自主 latent 判定 + 离线 LTI

形式化上，显式裁判写的是：

$$
[x,\texttt{<think>},c_1,\ldots,c_M,\texttt{</think>},y]
$$

LatentGRM 换成：

$$
[x,\texttt{<think>},z_1,\ldots,z_N,\texttt{</think>},y]
$$

其中 $x=(q,R,a,b)$，$y$ 是 A/B 偏好。

**Semantic Chunking。** 在 Latent-SFT 的词汇空间蒸馏框架上，把教师轨迹按 rubric 对齐的语义块切分：先按准则块分配固定压缩预算 $\rho$，再在块内用语言边界细化切点。相对固定速率分块，同一 $\rho=8$ 下，Semantic Chunking 在 Chat Hard / Precise IF / Focus 的 vote@5 上分别高 **2.2 / 3.9 / 1.7** 点，四域均值 +1.8。

**两阶段训练。** Stage 1 把显式评价压成 latent 目标并学 suffix 重建；Stage 2 让模型在推理时自主生成 latent 轨迹并预测偏好，不再依赖教师文本。

**LTI。** 在线路径只输出偏好；离线时用保存的 latent（含词汇概率表示）从评价开头重建整段文本，并按准则对齐教师评估，量重建保真度。

## 🧪 关键证据

**主结果（Table 1，vote@5 Avg.-8）。**

| 模型 | Avg.-8 | 相对显式对照 |
| --- | --- | --- |
| Rubric-RM-4B | 67.0 | — |
| LatentGRM-4B | 67.8 | +0.8 |
| Rubric-RM-8B | 68.1 | — |
| LatentGRM-8B | 69.0 | +0.9 |

最大优势在 Focus（+6.0 / +6.4）；Chat Hard、IFBench 两边都更偏 LatentGRM；RewardBench Chat 仍更偏显式 Rubric-RM；Precise IF 在 4B/8B 上方向不一致。PPE、RM-Bench Chat、HelpSteer3 差距不超过 0.4。投票从 vote@1 到 vote@5，LatentGRM Avg.-8 在 4B/8B 各涨约 2.2–2.3 点——**多条短轨迹也能投票。**

**推理成本（Table 3，vote@5，LatentGRM-8B vs Rubric-RM-8B）。**

| 域 | 步数压缩 | vLLM 加速 |
| --- | --- | --- |
| RB Chat | 8.87× | 6.15× |
| RB Chat Hard | 9.04× | 6.11× |
| RB2 Precise IF | 9.17× | 6.40× |
| RB2 Focus | 9.23× | 6.95× |

摘要里写的 8.9–9.2× / 6.1–7.0× 就是这组。等长对照强制两边都生成 50 步时，运行时差只剩约 1.017×，说明省的是步数，不是每步更便宜的魔法。

**重建（Table 4，607 条准则丰富样本）。** LTI 的 State Macro-F1 **98.93**、Vector Exact **93.98**，远高于 Prompt-only（40.20 / 18.61）与 Replay-hidden（69.42 / 33.03）。Prompt-only 的 ROUGE-L 还能到 63.36，却判错大量准则状态——**字面重合骗不了准则级检查。**

**泼冷水。** Chat 域显式裁判仍更强；LTI 重建主要在教师派生轨迹上验证，不等于自主 latent 每一步都同样可解释；干预与重建都依赖额外模型与对齐协议。

## 🔬 最有意思的部分

1. **rubric 既是压缩锚，也是探针。** 同一套准则结构同时决定切块边界、干预实验与重建对齐，把「压什么」和「验什么」绑在一起。
2. **潜源替换证明信息在轨迹里。** 固定基线 rubric，只换不同 rubric 下生成的 latent，准则干预源仍能把终点 log-odds 拉到 2.75，改写措辞对照则小得多。
3. **短轨迹也能投票。** 压缩后 vote@5–9 仍抬分，效率曲线整体左移（Figure 1），这和「越压越不能多样本」的直觉相反。
4. **和零样本多题筛查互补。** [JustAskJev](../../202609/week4/20260928_JustAskJev_论文解读_对齐失败检测别为每条准则再解码_一次问多题零样本就能筛.md) 想一次问多题少解码；LatentGRM 想把准则推理塞进连续状态。一个偏检测管线，一个偏奖励模型热路径。

## 🤔 我的判断

定位：**面向 generative reward model 的潜空间评价方法突破**，核心是用 rubric 引导的语义压缩把「多准则推理」从长文本解码里拆出来。

亮点三条：

1. 匹配数据与 backbone 下，4B/8B 都打平甚至略超显式 SFT 裁判，同时把轨迹与推理时间压一个数量级；
2. Semantic Chunking、准则干预、潜源替换、LTI 重建形成闭环，回答了「压完还剩什么」；
3. 开源仓库完整，便于接到已有 Rubric-RM / Latent-SFT 栈。

问题与局限：部分域仍输给显式裁判；自主轨迹的可解释性弱于教师轨迹重建；压缩率与 latent 预算如何按任务自适应未充分展开；投票仍要付多 rollout 成本。

对工程的直接启发：**(1)** 热路径只要偏好标签时，别默认生成整段评语，先试 latent 或短轨迹 + 投票；**(2)** 用 rubric 块做压缩边界，比固定速率切块更扛指令敏感任务；**(3)** 需要审计时把解释器放离线，别塞进每次在线判定；**(4)** 用准则级 State F1 / Vector Exact 查重建，别只看 ROUGE；**(5)** 和 [LLM-as-Judge](../../202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 的分层评测栈配合：昂贵显式裁判留在校准与抽检，latent 裁判扛高频筛选。

一句话收束：**LatentGRM 证明，多准则偏好判定不必逐字写评语——把 rubric 结构写进压缩边界，潜空间轨迹就能在压九倍之后仍保住竞争力和可探针的准则信息。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
