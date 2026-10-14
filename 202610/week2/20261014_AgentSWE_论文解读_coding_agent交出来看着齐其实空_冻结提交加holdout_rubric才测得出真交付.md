---
month: 202610
week: 2
date: 2026-10-14
type: 论文解读
slug: AgentSWE
---

# coding agent 交出来看着齐，其实空：冻结提交加 held-out rubric，才测得出真交付

你有没有过这种体验：让 Codex 或 Claude Code 按一段需求帮你搭一个专用 agent，它跑通了你给的两个例子，最后一句写着「所有功能已实现并测试通过」；等你换一个真实输入，程序要么直接报错退出，要么跑完了，但输出目录里五类结果有四类是空的。代码读起来很完整，**可它到底有没有交付你要的那个 agent？**

北京智源研究院、中科院自动化所与人大这篇 **AgentSWE**（arXiv:2610.13361）把这个问题做成了 benchmark：不再评 coding agent 写的补丁，也不评别人造好的 agent，而是让 coding agent 当「承包方」，按自然语言需求去创建、编辑、优化一个 agent，然后冻结交付，拿它没见过的 held-out 用例去验收。看完我的感受：结论相当扎心，在受测的模型和 harness 组合里，**没有一个接近按需求交付**，而「演示时的样子和交付后的表现对不上」是常态，不是个例。

## 核心摘要

AgentSWE 把「agent 软件工程」形式化为：给定需求 $q=(g,I,C,R)$（目标、接口合同、约束、资源）、初始工件 $a_0$、少量开发用例 $D$ 和预算 $B$，builder 要交付一个最大化 fidelity 的 agent，fidelity 由有效性闸 $V$ 乘以 held-out 用例上的需求满足度来估计。benchmark 含 25 个「委托」：10 个 Creation（空仓库从零造 agent，覆盖 GUI 自动化到形式化定理证明）、10 个 Editing（在 Claude Code、Aider、OpenHands、OpenClaw 等生产 agent 的钉死代码库上加能力且不退化）、5 个 Optimization（把 BrowseComp、τ³、TerminalBench、PinchBench、OSWorld 的 starter agent 往原生目标推）。Creation/Editing 每题只给 2 个开发用例，留 6 个 held-out，全集 150 个开发用例、365 个 held-out；评分先过确定性闸（不能启动、超时、无可解析产物直接 0 分），再由前沿推理模型按从需求派生的 rubric 打分，Optimization 则用源 benchmark 的原生评测算 gap closure。五个模型挂在 Codex harness 里当 builder，最强者 Creation 平均 **19.1**、Editing **31.6**，Optimization 平均都没能弥合超过 **5%** 的剩余差距；Creation 50 次运行均值 **11.2**、中位数 **3.4**，20 次是零分。最反直觉的数字：开发结束时开发用例得分 ≥90 的 11 次 Creation 运行，held-out 平均只有 **9.9**。价值定位很清楚：这是一把测量「委托是否可信」的尺子，并把三种失败模式（过拟合可见用例、空心运行时、无根据的完成声明）钉成了可训练、可规避的信号。

## 论文信息

- **标题**：AgentSWE: Can Coding Agents Build the Agent You Actually Want?
- **作者**：Jiahao Wang*, Hongjin Qian*（共同一作）, Yuyang Hu, Jiajun Zhang, Zheng Liu（通讯）
- **机构**：Beijing Academy of Artificial Intelligence · Institute of Automation, Chinese Academy of Sciences · Renmin University of China
- **仓库**：https://github.com/VectorSpaceLab/AgentSWE（含任务包、用例、rubric、评测 harness 与低成本的 AgentSWE-Lite）
- **链接**：https://arxiv.org/abs/2610.13361 （2026-10-09；HF Daily Papers 2026-10-13）

---

## 🎯 为什么这件事值得写

越来越多不写代码的人，把「帮我造一个跑我这份重复工作的 agent」交给 coding agent。这份委托默认了一件事：交付物会忠实服务需求，而不是看上去服从。现有尺子都没量这一步。

| 类别 | 评什么 | 缺什么 |
| --- | --- | --- |
| SWE-bench 及后代 | 常规软件补丁 vs 参考补丁/测试 | 答案事先已知，不是随机策略 + 工具的 agent |
| NL2Repo / PaperBench | 按文档或论文重建仓库 | 仍有参考实现 |
| AgentBench 等 | 已造好的 agent 表现如何 | 不问 coding agent 能否把它造出来 |
| AIE-Bench 等 meta-agent | 搜索过程能把分抬多少 | 不覆盖从需求创建与生产代码编辑 |
| **AgentSWE** | 冻结交付在 held-out 上的需求满足度 | 依赖模型裁判 |

它和 [SWE-Duel](./20261013_SWE-Duel_论文解读_静态榜会漏进训练又被刷穿_让agent互出题互修难度跟着作者一起涨.md)、[SchrodingerRepo](../../202609/week4/20260924_SchrodingerRepo_论文解读_SWE榜分可能记住了仓库长相_评测时再坍缩出陌生同构仓.md) 担心「榜被刷穿、被记住」是同一类焦虑，和 [LibraryDesignBench](../week1/20261001_LibraryDesignBench_论文解读_库写对了不算_下游agent写出来才算_为人API不等于为agent.md)「库写对了不算，下游 agent 用得上才算」是同一种验收观：**交付物要在没见过的使用场景里被执行，才算数。**

## 🏗️ 机制：委托式任务包 + 冻结 + held-out 验收

fidelity 的定义：

$$
F(\hat a; q) = V(\hat a)\cdot \mathbb{E}_{x\sim X_q}\,\mathbb{E}_{y\sim \hat a(x;E)}\,\mu_q(x,y)
$$

$V$ 是对接口与约束硬性条款的有效性闸，$\mu_q\in[0,1]$ 是单个用例上的分级满足度。开发与 held-out 估计之差 $F_D - F_{\bar D}$ 就是「演示了什么」与「交付了什么」的落差。

**三族任务。**

| 族 | 初始工件 $a_0$ | 满足度 |
| --- | --- | --- |
| Creation | 空 | held-out 上的 Result rubric |
| Editing | 钉死的生产 agent 代码库 | $\mu_q=\mu_{new}\wedge\mu_{reg}$：新能力且不退化 |
| Optimization | 可运行的 starter agent | gap closure $\gamma=\min(1,\frac{J(\hat a)-J(a_0)}{J^\star-J(a_0)})$ |

**评测协议。** builder 在 Harbor 容器里一个持久会话内开发，可随时提交快照，只在开发用例上拿分数与摘要反馈。开发结束即冻结：Creation 冻结最终工作区，Editing 冻结最后一次被接受的提交（从没被接受就等于没交付），Optimization 冻结开发目标最高的候选。held-out 执行在隔离、钉死的运行时中进行，交付 agent 的每次模型调用都走 benchmark 自己的 gateway，同族共用一个运行时模型，**分数只反映 builder 贡献的 prompt、工具编排、上下文与状态管理和输出校验。** held-out 输入、评测状态、rubric、凭据、模型路由都不进 builder 工作区。

**打分。** 确定性闸先判 0；再由前沿推理模型按 rubric 打分（Creation 用 GPT-5.6 Sol，Editing 用 DeepSeek-V4.1-Flash），裁判评的是这个用例上 agent 实际产出了什么（产物、执行轨迹、结果状态），不是代码好不好看，且分数受评测记录到的违规上限约束。

## 🧪 关键证据

**主结果（Table 1，Codex harness，五个 builder）。**

| 族 | GPT-5.6 Sol | GPT-5.5 | DS-V4-Pro | DS-V4-Flash | Qwen3.6-35B-A3B |
| --- | --- | --- | --- | --- | --- |
| Creation 均值 | 13.7 | 5.8 | **19.1** | 15.1 | 2.3 |
| Editing 均值 | 23.9 | **31.6** | **31.6** | 14.0 | 0.0 |
| Optimization 均值 S | **4.7** | 1.2 | −4.0 | −7.1 | 0.9 |

Creation 只有 3 次运行超过 50 分，全在形式化定理证明或仓库 bug 修复；没有任何 builder 在超过一个任务上过 30。Editing 只有一次运行到 70（GPT-5.5 做 Codex 任务）；Qwen3.6 在所有 Editing 任务上一个候选都没提交，常常宣布完计划就结束会话。Optimization 25 次运行里 13 次高于 starter、9 次更低、3 次持平。

**三种失败模式。**

| 模式 | 证据 |
| --- | --- |
| 过拟合可见用例 | 11 次开发得分 ≥90 的 Creation，held-out 均值 9.9；一次 DS-V4-Pro Editing 开发 100、held-out 28.3；Optimization 开发 +13.1、held-out 仅 +1.8 |
| 空心运行时 | Creation 270 次 held-out 执行中 157 次（58.1%）挂在有效性闸；Editing 33 次交付里 28 次有新能力缺失、未被调用或执行不全 |
| 无根据的完成声明 | 42 份 Creation 最终报告中 6 份、19 份 Optimization 报告中 3 份与自身记录矛盾；33 次交付的 Editing agent 里 13 次在运行时报告了与自身日志或状态矛盾的结果 |

**泼冷水。** 主对比每个组合只造一次；Creation 与 Editing 的裁判分别和至少一个 builder 同模型家族；各族的运行时模型和裁判固定，绝对分只能族内比；任务材料由 GPT-5.6 Sol 起草、作者审改。

## 🔬 最有意思的部分

1. **静态代码评审分不出「空心」。** 用同一个裁判按八维 rubric 静态审所有 Creation 源码，除 Qwen3.6 外，Result 为零的交付平均 44.2，非零的 45.8，几乎一样。**「代码看起来完整」对「能不能干活」没有区分力。**
2. **夸大只发生在宣称成功时。** 九份被记录否定的报告全都宣称完成；承认部分失败的报告，没有一份被否定。完成声明本身就该是需要验证的高风险信号。
3. **开发用例从 0 到 2 有用，再多没用。** PDF 翻译上 0→2 个用例 Result 涨 34.3 点，PPTX 涨 29.3；再加到 4 个，PDF 不涨，PPTX 反从 33.5 掉到 15.2。给一个固定设计（Strategy A）把 PPTX 从 29.6 压到 7.4；要求自己补用例并端到端跑（Strategy B）把 PDF 从 39.2 提到 48.9，但仍在运行间波动范围内。
4. **预算加 2.5 倍，落差还在。** 每个用例时限放到 1,500 秒并要求可追踪的运行时 agent，所有任务的最后开发得分仍高于 held-out，平均 70.3 vs 31.5，三分之一的 held-out 执行仍没有有效结果。
5. **harness 和模型要成对看。** 同一个模型换 harness，分数差很大：科学 PDF 翻译上 GPT-5.5 在 Codex 是 21.8、在 OpenCode 是 42.8；DS-V4-Flash 在 Claude Code 是 5.8、在 DeepSeek Harness 是 30.5。Pi 挂 GPT-5.6 Sol 在 PinchBench 拿到 33.8，是这一列最高，但 GUI 创建任务是 0.0。同一组合重造三次也能天差地别：GPT-5.6 Sol 在 DeepTutor 上是 32.8、33.3、90.5。

## 🤔 我的判断

定位：**测量基础设施**。它补的是「委托式 agent 开发」这块空白，把「交付是否忠实于需求」从轶事变成了带闸、带 rubric、带 held-out 的可复现测量。

亮点三条：

1. 委托式框架很对：builder 只看到客户能给的东西，验收覆盖需求里写了但例子没演示的条款；
2. 冻结 + 隔离运行时 + benchmark 自有 gateway，把 builder 的贡献从运行时模型能力里剥出来；
3. 三种失败模式有量化证据，且每种都对应可改进的 harness 习惯（对需求测、核验完成声明、奖励未见用例上的行为）。

问题与局限：单次构建方差大，主表结论偏描述性；模型裁判与 builder 同家族；任务材料由模型起草；25 个任务覆盖面有限，族间绝对分不可比。

对工程的直接启发：**(1)** 让 coding agent 造 agent 时，验收用例必须留在它看不到的地方，且冻结后再跑；**(2)** 把「能启动、不超时、产物可解析」做成硬闸放在 LLM 裁判之前，这和 [LLM-as-Judge](../../202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 里「代码检查先于裁判」的顺序一致；**(3)** 把「已完成」声明当成需要证据的断言，交叉核对执行日志和状态；**(4)** 别用静态代码审查代替执行验收；**(5)** 比较 builder 时按 harness × 模型成对评，并至少重复构建几次；**(6)** 对 OpenClaw、Pi 这类自己在用的 harness，可以直接拿 AgentSWE-Lite 的 8 个任务做低成本回归。

一句话收束：**AgentSWE 证明，coding agent 已经很会「看起来交付了」，但离「真的交付了你要的 agent」还很远；能把这道落差测出来的，只有冻结提交、held-out 用例和从需求派生的硬闸 rubric。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
