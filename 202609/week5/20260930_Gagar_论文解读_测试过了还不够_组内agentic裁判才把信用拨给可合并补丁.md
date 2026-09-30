---
month: 202609
week: 5
date: 2026-09-30
type: 论文解读
slug: Gagar
---

# 测试过了还不够：组内 agentic 裁判，才把信用拨给可合并补丁

你有没有过这种体验：用 GRPO 训 coding agent，同一题 16 条 rollout 里好几条都过了测试——有的补丁精准、贴仓库惯例，有的却顺手改一堆无关文件、关掉校验、或轨迹绕了上百轮才碰巧绿。二进制 reward 下一律 \(1-\bar R\) 的正优势，**质量差异在训练信号里直接消失**。小米 LLM Core 这篇 Gagar（arXiv:2609.32577；Groupwise Agentic Grading for Advantage Redistribution）把问题钉死：among successful trajectories, which ones deserve stronger reinforcement？看完我的感受：这正好接在 Byron 的 **LLM-as-judge / code-agent RL / 信用分配** 线上——比「再加一个稠密 PRM」更狠的是 **组内联合检视 + 保正优势总和的再分配**；工业尺度还直接喂进了 MiMo-V2.6 Flash/Pro 的混合任务 RL。

## 核心摘要

Gagar = **组内 agentic 打分** + **保和式优势再分配**。动态采样只保留「有过有挂」的 mixed-outcome 组；把任务说明、仓库、完整轨迹、补丁与测试日志放进共享 workspace，由 SFT 训出的 agentic grader（Flash/Pro 实验里统一用 MiMo-V2.6-Pro 的 pre-RL SFT 当在线裁判）联合检视、跑定向检查，再按五维质量（方案合适度 / 实现精度 / 改动最小化 / 副作用 / 与代码库一致性）给过测候选分档排序并映射折扣因子 \(f_i\)。关键不是「把烂过测直接压低」——单纯 downweight 会让正优势总和变小、负优势相对变强，训练熵与轨迹长度爆炸；Gagar 先按 \(f_i\) 压权，再以 \(\lambda=S_+/\sum f_j A_j\) **把正优势总和 \(S_+\) 补回**，失败轨迹优势不动，过测之间的相对比 \(f_i/f_j\) 保持。Code-only Flash（310B total / 15B active）上相对二进制基线：DeepSWE v1.1 同步 +12.1 pp（62.2% vs 50.2% @step 28）、峰值 63.4%；盲评质量胜率约 69.8%；回合/token 同步下降。混合任务 RL 后 Flash / Pro 达 DeepSWE 67.9 / 71.9。已进 MiMo-V2.6 系列大规模 RL 管线。

## 论文信息

- **标题**：Groupwise Agentic Grading and Advantage Redistribution for Code Agent RL
- **作者**：Jinhao Dong、Liang Zhao、Zihao Yue、Wenhan Ma、Linghao Zhang、Lei Li、Shicheng Li、Yifan Song、Bowen Ye、Fuli Luo†（† correspondence）
- **机构**：LLM Core, Xiaomi；人大 / 北大 / 港大
- **链接**：https://arxiv.org/abs/2609.32577 · https://arxiv.org/pdf/2609.32577 · https://arxiv.org/html/2609.32577
- **观察时间**：2026-09-30（HF Daily Papers；接续 week5 裁判/信用主题）

---

## 🎯 为什么这件事值得写

Coding-agent RL 已经默认「可执行测试 = 可验证奖励」。但 GRPO/Dr.GRPO 在组内对所有过测轨迹给**相同** outcome advantage——它回答了「会不会做」，没回答「做得好不好、能不能合并」。开发者体验差的补丁（范围外修改、弱化校验、策略绕远）仍拿到满额正强化。

和本周已归档的 [SLCA-GRPO](./20260929_SLCA-GRPO_论文解读_工具段和总结段别共用一个优势_分段锁信用.md)（切 tool/summary 段）不同，Gagar 切的是 **同任务多条过测实现之间的质量轴**；和 [DRM](./20260929_DRM_论文解读_奖励别再点估计_扩散头才装得下多峰偏好.md) 的「奖励分布」互补——这里裁判是 **可交互检仓库的 agentic grader**，不是再训一个标量 reward head。对 Byron：这是「binary 测试先过门 → 再上结构化 / 组内 judge」建设顺序的工业样本。

## 🏗️ 机制：联合检视 → 三档折扣 → 保和再分配

**训练设定。** 任务 \(x\) 上采 \(n\) 条有效轨迹，有效奖励 \(R_i\in\{0,1\}\)（确认 hack / 泄露答案 → 置 0 当失败）。用 mean-centered \(A_i=R_i-\bar R\)；只保留 \(\mathcal{P}\) 与 \(\mathcal{F}\) 皆非空的组。

**Groupwise Agentic Grading。** Grader 不是单次 chat 打分：先看回合摘要，再按需读轨迹片段、对照补丁与测试日志，必要时跑定向检查；负面结论必须引用补丁位置 / 轨迹事件 / 执行结果。过测候选进三档 \(\mathcal{T}_1/\mathcal{T}_2/\mathcal{T}_3\)（强 / 中 / 重大缺陷），档内按加权准则排序，映射 \(f_i\in(0,1]\)。工程上：Claude Opus 5 初版约 2000s/组；换成 Pro SFT grader ~600s/组，并与 partial-rollout 异步重叠。

**Sum-Preserving Redistribution。** 只做 \(\widetilde A_i=f_i A_i\)（\(i\in\mathcal{P}\)）会制造正优势赤字 \(D=\sum(1-f_i)A_i\)，组内和变为 \(-D\)——负侧相对更强。Gagar 令
\[
\lambda=\frac{S_+}{\sum_{j\in\mathcal{P}}f_j A_j},\qquad
A_i^\star=\begin{cases}\lambda f_i A_i,& i\in\mathcal{P}\\ A_i,& i\in\mathcal{F}\end{cases}
\]
从而 \(\sum_{\mathcal{P}}A_i^\star=S_+\)、失败侧不变、过测相对比 \(A_i^\star/A_j^\star=f_i/f_j\) 保留。等价写法：\(A_i^\star=(1-\bar R)\,f_i/\bar f_{\mathcal{P}}\)——高于过测均值的因子「吃」信用，低于的「吐」信用。单过测或全平局时退回原 outcome advantage。**不需要**跨轨迹对齐中间状态——这对 >100 轮、仓库状态发散的 coding 轨迹至关重要。

## 🧪 关键证据

**Code-only Flash vs 二进制基线。** 基线 DeepSWE 从 step 20 的 58.5% 崩到 step 28 的 50.2%（作者因此停训）；同步 Gagar 62.2%（+12.1 pp），继续训峰值 63.4%@44。SWE-bench Pro：基线约 59% 平台期，Gagar 到 step 52 达 62.5%。效率：DeepSWE 同步回合 132.3→111.6（−15.6%）、token 191.9k→172.9k（−9.9%）；SWE-bench Pro 回合/token 亦降。

**质量盲评（Claude Opus 5，非在线裁判）。** 30 题 DeepSWE 抽样：质量分 4.03 vs 3.70；过测间胜率 69.8%；组内第一 65.0%；\(\mathcal{T}_1\) 占比 25.4%→34.3%。增益集中在精度、最小化、副作用。

**消融：只压权不保和。** 熵 0.359→0.905、训练长度 47.1k→114.1k；DeepSWE 抖动（56.5→48.8→56.2），回合/token 飙到 158/246k 量级；完整再分配熵更缓、同步 DeepSWE 62.2%。**「质量偏好」若破坏正负信用平衡，会比没裁判更糟。**

**混合任务工业 run。** 1568 prompts × 16 rollouts；Flash/Pro DeepSWE 67.9/71.9，Pro SWE-bench Pro 62.7（高于文中 GPT-5.6 Sol 的 60.5）。

**泼冷水。** 在线裁判是同一家族 Pro SFT，外部泛化未充分拆；质量盲评仍是 LLM panel；hack 检测依赖「确认」流程；非 coding 域的组内质量定义未展开。

## 🔬 最有意思的部分

1. **失败轨迹不当「质量榜」，当上下文。** 挂测样本暴露漏需求与坏策略，只给过测排序——和「把失败也塞进同一标量排序」不同。
2. **Agentic ≠ 更贵的点估计。** 价值在共享 workspace 里的**对比与定向执行检查**；单候选静态打分会漏「不必要复杂度」。
3. **保和是稳定性旋钮，不是美学。** 消融把「压烂补丁」和「训练炸」解耦：缺的是正优势守恒。
4. **与时间轴 / 段轴正交。** 可叠加 SLCA 式段锁或过程监督，但本文证明**实现级组内重排** alone 已足够改变后期动力学。
5. **延迟工程是一等公民。** 600s grader + 异步重叠——提醒 harness：judge 若不进调度，再好的公式也上不了工业 batch。

## 🤔 我的判断

定位：**coding-agent RL 的过测内质量信用层**（组内 agentic grader + 保和优势再分配），工业验证到 310B/1T 级 MiMo。亮点三：

1. 把「过测同权」诊断成可修复的信号缺失，而不是抱怨测试不够严；
2. 保和再分配把质量偏好从「削弱成功」改成「在成功子集内零和调仓」；
3. 效率与 pass rate 同向——不是用更长轨迹换分。

局限：裁判家族耦合、盲评仍贵、非代码域准则要重做。对 Byron：若你在跑 DeepSWE/SWE 类 GRPO，**默认别让所有 pass 同分**——最小落地可以是：同题过测补丁做 diff 体积 / 无关文件触碰 / 测试删改的 **程序化 rubric**，再考虑上 agentic grader；任何 \(f_i\) 压权后务必 **补回 \(S_+\)** 或显式重居中。评测侧加「过测内质量 / 回合数」切片，别只报 pass@k。接 judge 建设顺序：可执行测试 → 程序化质量门 → 组内结构化 grader → 再谈昂贵多裁判校准（DIAL/JEV）。

一句话收束：**绿测只说明能骗过 CI，不说明该被强化**——Gagar 让组内裁判决定「哪条绿更值得涨工资」，并用保和避免把工资总额砍掉。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
