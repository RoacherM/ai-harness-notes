---
month: 202610
week: 3
date: 2026-10-16
type: 论文解读
slug: Cortex
---

# 加一条 skill 别靠一次成功：测量门控，才留下 rules

你有没有过这种体验：Claude Code 又写歪了项目约定，你随手往 `.claude/rules/` 或 skills 里塞一行「下次别这样」；看着这次过了，就当它永远有用——可同一任务再跑一遍，有时过有时不过，上下文账单却每回合都在付。**一次成功，凭什么证明该留住这条 harness 文本？**

独立研究者 Samuele Giampieri 这篇 **Cortex**（arXiv:2610.14216）把答案写成工具：会话结束 `/harvest` 把「坏提交→修好」收成可回放任务并记下反复纠正；`/evolve` 每次只提一条 skill/rule，在有/无候选上各跑多轮，**只有整体有增益、至少开火一次、且没有单任务明显变差**才留下；`/prune` 反向拆件重测，没见过开火的绝不删。看完我的感受：这和 [ReHarness](../week2/20261012_ReHarness_论文解读_反思写进记忆就不再改_反向修订让harness状态也能吃到下游反馈.md)、[MetaSkill](../../202610/week1/20261002_MetaSkill_论文解读_别把技能塞给Target_教Builder学元技能才配得动harness.md)、[EvoSkill-GUI](../../202609/week3/20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) 同属「harness 文本要进化」，但 Cortex 把门控钉在**曝光感知的测量语义**上——Haiku 4.5 上 held-out 规则族 **+55.0** 点，换到已会守约的 Sonnet 5 只剩 **+5.6**，而且留下的几乎全是 path-gated **rules**，不是 always-on skills。

## 核心摘要

Cortex 面向 Claude Code 的 harness（CLAUDE.md + skills + rules），核心原则：**只在文本真正进入 agent 上下文的地方计功过**。`/harvest` 不跑 rollout，只把修好的会话固化为（broken commit, request, check）任务并追加每次纠正为 lesson；`/evolve` 找复发主题、写一条候选并选加载层（path-gated 优先），在可弃克隆上对 live vs live+candidate 做 sweep，保留条件为有净增益、曾开火（rule 载入或 skill 被打开）、且不超过公差的单任务伤害（live 从未失败的任务公差为 0）；`/prune` 逐件移除重测，未见开火则不删。评测：Claude Haiku 4.5 既当被测 agent 也跑 Cortex；合成仓四条「房子规矩」任务族 + 一条无规矩可学的控制族；四次独立 run（R1–R4）各 10 轮脚本会话，再在 **30** 个从未见过的 held-out 场景上量。规则族相对「无 skills/rules」基线 **+55.0** 点（95% CI [+34.1, +74.1]，置换 $p=0.0001$），控制族 **−0.8** 且无人保留控制项；留下的 18 项中 **16 是 rules**，全部带 paths，不增加每回合必读文本（除共享的 559 字符 CLAUDE.md）。Placebo 20 / 有害 5 无一保留，但作者坦承：无用/有害 skill 多半从未被打开，有害 rule 的 screen 设计也难测到伤害——证明的是「不用则不留」，还不是「能抓破坏」。R1 harness 在 Haiku 上 **+66.7**，迁到 Sonnet 5 仅 **+5.6**（CI 含 0）；真实开源仓因很少纠正而学不到东西。价值定位：给「印象策展的 harness」换成可复现的测量门控进化/修剪环。

## 论文信息

- **标题**：Cortex: Measurement-Gated Evolution and Pruning of Coding-Agent Skills and Rules
- **作者**：Samuele Giampieri
- **机构**：Independent researcher
- **仓库**：https://github.com/samugit83/cortex （评测快照 tag `v1.0-eval`；数据与复现脚本 tag `v1.0.5`）
- **链接**：https://arxiv.org/abs/2610.14216 （2026-10-09）

---

## 🎯 为什么这件事值得写

Harness 文本（项目说明、可选 skill、按路径自动载入的 rule）改起来只要一分钟，但「有没有用」常靠眼缘。Agent 成功是噪声量：同一 harness 两次测量可差 **−3.3** 点（95% CI [−7.3, 0.0]）——看见一次提升就会当成效应。已有工作报告加 skill 可能变差、仓库内增益与跑间噪声分不清、弱模型甚至激活不了新增文本。自动进化 prompt/agent/harness 的系统多对齐公开榜；开发者面对的是**一个仓、自家约定、少量自有任务、固定上下文预算**。

Cortex 的切口很窄：不为刷榜搜最优壳，而是回答「这条纠正该不该写进仓库、写了该不该删」。

| 策展方式 | 证据 | 风险 |
| --- | --- | --- |
| 看一次成功就加 | 轶事 | 噪声当因果；上下文永远涨 |
| 只加不测 | 无 | 无用/有害并存 |
| 下游总分优化 | 榜分数 | 和本仓约定脱节 |
| **曝光感知的有/无对照** | 多次 rollout + 是否开火 | 测得到才留/删 |

## 🏗️ 机制：harvest → evolve → prune，同一套曝光语义

**加载层（Claude Code）。** CLAUDE.md 每会话必读；rule 可带 `paths`，匹配文件被读时自动进上下文；skill 默认只挂描述，正文要 agent 调 Skill 工具才打开——无 paths 的 skill 每回合都出现在列表里（always-on），有 paths 的只在摸到相关文件时列出（path-gated）。

**`/harvest`。** 问两件事：这会话有没有从坏到好？有没有纠正？固化可回放任务（含证明能修的 fix.patch）并写 lesson；不写 skill/rule。

**`/evolve`。** 每周期一条候选；先廉价 screen（只跑候选相关且 live 并非总过的任务），再全量 sweep；score.sh 出裁决。KILL 进 graveyard 并记原因。CLAUDE.md 改动可记 journal 但不走同一套测量门（门控只管 skills/rules）。

**`/prune`。** 计划需人批；在能加载该件的任务上比「有 vs 无」；ACCEPT 删、REJECT 留、UNMEASURED（从未开火）按设计不删——与 evolve「从未开火不留」对称。

**曝光原则。** 功过只计开火子集；开火未知则阻塞裁决。有害性只能在「无该项也会过」的任务上显形。

## 🧪 关键证据

**Held-out 主效应。** 四规则族：evolved vs 空 harness **+55.0** 点（CI [+34.1, +74.1]，$p=0.0001$）；控制族 −0.8，无控制项被留。R1 上 evolved 比「把贡献指南（写明房子规矩）追加进 CLAUDE.md」高 **11.1** 点（CI [0.0, +22.5]，触零且跨 batch，需谨慎读）。

**加载形态。** 跨 run 保留 18 项，**16 为 rules**；全部带 paths，不增加 always-on 正文。Gate 测试：path-gated skill 列出 66 次、打开 **28** 次；always-on skill 列出 **198** 次、打开 **0** 次——「挂在每回合」几乎等于没被读。

**已知答案管道（弱于标题数字）。** 20 placebo + 5 有害无一 KEEP。但：placebo 与 2 个有害 skill 在 screen 上从未打开→因未使用被拒或无法显示增益（12 placebo 拒用，8 无裁决）；3 个有害 rule 会加载，可 screen 专挑 live 已失败的任务，无法展示「无该项会过」上的伤害，故无裁决——甚至出现有害 rule 打挂「无它会过」的任务仍无裁决的屏幕。结论作者自己写清了：**证明不用则不留，尚未证明能抓破坏。** 已知好候选仅 **2/4** 留下。

**模型迁移。** R1 harness：Haiku **+66.7** → Sonnet 5 **+5.6**（CI [−2.8, +13.9]）；升级后一轮 `/prune`（每项有/无各 3 次）标出 4 项中 2 项可删。价值约等于「模型还需要被纠正多少」。

**`/prune` 缺口。** 8 项测试中 6 项裁决符合预期，但 4 个植入项里只对 2 个对；曾留下「自己 sweep 从未打开」的无用 always-on skill，因为去掉后 2 个任务通过率下降；从未开火的 always-on skill 在需要 UNMEASURED 的设定下会被计划直接排除，于是不测就留下。

**真实仓。** 开源项目上 agent 很少被纠正→无复发→环学不到东西。可选外部 judge 在 run 中关闭，事后回放也未预判裁决。

**泼冷水。** 合成仓 + 脚本开发者，外部效度有限；A/A 噪声已达数点，单次「看起来涨了」不可信；有害检测未闭合；known-good 召回低；prune 对未开火 always-on 仍可能放行；强模型/少纠正场景 ROI 接近零；可选 judge 尚未证明有用。

## 🔬 最有意思的部分

1. **交付方式比文案内容更决定命运。** Rules 自动进上下文，skills 要被选——保留集几乎全是前者；always-on skill 列表刷屏却 0 打开。
2. **+55pp 与 Sonnet +5.6 必须绑在一起读。** Cortex 不是「无脑涨分插件」，而是「弱模型 + 可学约定」时的纠正放大器。
3. **曝光语义统一了加与删。** 同一把尺：没进上下文就不领功、也不背锅——避免「没加载却归因」的假因果。
4. **Placebo 清零的标题需要脚注。** 管道拒的是未使用，不是证伪有害；工程上仍要人审高风险 rule。
5. **和 Skill-Use / Subagents-vs-Skills 对话。** 仓库已有「挂上 ≠ 会用」；Cortex 用打开率直接把 always-on skill 判成测量上的死信。

## 🤔 我的判断

定位：**工程样本 + 测量语义贡献**。不一定是最大幅度的 harness 搜索器，但把「该不该留一句纠正」变成可复现实验，这步对日常 Claude Code 用户比刷公开榜更贴肉。

亮点三条：

1. 曝光感知的加/删对称门控；
2. held-out + 控制族 + A/A，把噪声和真增益分开讲；
3. 明确报告失败模式（有害检测、prune 漏洞、强模型/真实仓 ROI）。

问题与局限：合成评测；有害未测死；known-good 召回；prune 对未开火件；依赖纠正密度；Claude Code 加载语义绑定。

对工程的直接启发：**(1)** 别因一次成功就 commit skill/rule——至少有/无对照多跑几轮；**(2)** 优先写 **path-gated rules**，慎用 always-on skills（列表可见 ≠ 正文读入）；**(3)** 进化环要记录「是否开火」，没曝光就不要归因；**(4)** 换更强模型后重跑 `/prune`，旧 harness 可能过期；**(5)** 与 [ReHarness](../week2/20261012_ReHarness_论文解读_反思写进记忆就不再改_反向修订让harness状态也能吃到下游反馈.md)、[Skill-Use](../../202610/week1/20261002_Skill-Use_论文解读_技能挂上了不等于会用_Trigger合规边界才测真使用.md)、[MetaSkill](../../202610/week1/20261002_MetaSkill_论文解读_别把技能塞给Target_教Builder学元技能才配得动harness.md) 叠用：纠正要能回流、技能要能量使用、元技能要会配 harness——Cortex 补的是**留下之前的测量闸**。

一句话收束：**harness 文本便宜到可以随手加，也正因如此更需要门控；Cortex 证明，测量过、开过火、且没砸任务的 path-gated rules，才配留下来——一次成功从来不够。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
