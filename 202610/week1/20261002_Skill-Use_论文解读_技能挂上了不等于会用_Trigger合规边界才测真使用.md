---
month: 202610
week: 1
date: 2026-10-02
type: 论文解读
slug: Skill-Use
---

# 技能挂上了不等于会用：Trigger / Compliance / Boundary，才测「真·技能使用」

你有没有过这种体验：Claude Code / Codex / Cursor / OpenClaw 技能目录越堆越长，README 写着「progressive disclosure」，可 agent 要么根本不打开 SKILL.md，要么打开了仍抄近道、踩禁区——任务「看起来做完了」，技能契约早被撕掉。腾讯混元 / 华师大 / 港科大 / 复旦这篇 **Skill-Use**（arXiv:2608.04828）把缺口说透：现有榜多评技能质量或最终成败，**几乎不评「agent 能否自己认出并忠实用技能」**。看完我的感受：这是 skills eng 的**过程级测谎仪**——最强配置 SU 仅 **0.613**，且 **换 harness 排名会翻**。

## 核心摘要

S KILL -U SE 在 progressive disclosure 下评测：agent 起初只见技能 **名 + 短描述 + 路径**，须主动检索全文再遵循。三维拆解——**Trigger**（是否 Retrieve 目标技能）、**Compliance**（规定步骤加权达成）、**Boundary**（禁止操作是否缺席）；门控总分 \(\mathrm{SU}=g\cdot[\alpha C+(1-\alpha)B]\)（\(\alpha=0.7\)，**未触发则执行分不计**）。基准：**79** 条真实社区技能 × **177** 可执行任务 × 九域；每任务落在隔离 **Docker** 沙箱、真实文件资产；**1314** 条轨迹级 rubric 项（均 7.4，技能独占要求均 4.8）。八模型 × 两 harness（**Claude Code vs Codex**）：最强为 GPT-5.5@CC 的 **SU=0.613**；触发后最高 Compliance† 也只 **0.638**；Boundary 普遍高于 Compliance（安全合规类例外）。CC 下部分开源中坚 Trigger≈0.32–0.34 但触发后合规不差——缺的是**认出该用**；Codex 抬弱 Trigger、却拉大条件合规差距（GPT-5.5 的 Compliance† 掉超一成）。结论：**技能使用是 model–harness 配置属性，不是模型固定能力。** 仓库：JinyiHan99/Skill-Use-Bench。

## 论文信息

- **标题**：Skill-Use: Can LLMs Actually Use Skills in Agentic Harnesses?
- **作者 / 机构**：Jinyi Han, Yuanjian Xu 等；通讯 Zhichao Hu, Yanghua Xiao（Tencent Hunyuan · ECNU · HKUST · Fudan）
- **仓库**：https://github.com/JinyiHan99/Skill-Use-Bench
- **链接**：https://arxiv.org/abs/2608.04828 · https://arxiv.org/pdf/2608.04828
- **观察时间**：2026-10-02（skills · progressive disclosure · CC/Codex harness 条件化）

---

## 🎯 为什么这件事值得写

Skills 已成 coding-agent 工具链标配，但生态默认假设「挂上技能 ≈ 会用技能」。SkillsBench 类工作看边际成功率；SLBench 预加载全文，跳过检索；指令遵循榜把约束写进 prompt 原文——都**绕开了 progressive disclosure 的认路过程**。Skill-Use 把「认出 → 照做 → 守界」拆开，并用 Docker 轨迹 rubric 打分，直接服务 Byron 的 **skills eng · 双 harness（CC/Codex）· LLM-as-judge** 三角：没有过程门控，星标再多的技能库也可能是装饰。

## 🏗️ 机制：门控 SU 与三段构造管线

实例 \(x=(s,q,E,R)\)：技能 \(s=(m,p)\)（元数据 / 全文）、任务、沙箱、隐藏 rubric。轨迹含读文件、调工具、改制品；Trigger 是指示函数「\(\mathrm{Retrieve}(s)\in\tau\)」。构造三阶段：（1）从 ~10.5 万候选筛真实技能（技能独占性 + 过程可观测），masked-skill 去掉「只看题就能做对」的假要求；（2）三层 rubric：可评门控 / 技能导出（进 SU）/ 任务结果（**不进 SU**）；（3）端到端验证可执行性、跨组件一致、rubric 回放稳定性；多代理对抗审查修订范围与可验证性。LLM judge（GPT-5.4，T=0）仅在规则写不死时、且**范围受限**。

预加载对照：把全文塞进初始指令——主要抬 Trigger，弱 Trigger 模型增益最大；说明 native 模式下**检索本身是主瓶颈之一**。

## 🧪 关键证据

**主表（Table 1）摘点。** CC：GPT-5.5 SU **0.613**（Trigger 0.972 / C 0.611 / B 0.718）；Claude Opus 4.7/4.8 ≈0.599/0.597；DeepSeek-V4-Pro / GLM-5.1 / Kimi-K2.6 Trigger 仅 **0.324/0.324/0.337**，SU 跌到 0.19–0.27，但 Compliance† 可到 0.58–0.64。Codex：榜首变 Claude Opus 4.8（**0.559**）；GPT-5.5 掉到 **0.503**（Compliance† 0.490）；若干中坚 Trigger 明显回升。两 harness 的 per-model SU 相关仅中等——**排名不可直接迁移**。

**技能类型。** 文件处理 / 数据库 / 数据科学合规更高（工具与格式具体）；商业分析、软件架构偏低（开放规划）。Boundary>Compliance 几乎全域；唯独 Security & Compliance：步骤做得像，却在该克制时仍调用——**守界比照做更难**的反例。

**注入模式。** Preload 相对 native 的 ΔSU：抽象命名技能受益最大（认不出来），关键词丰富的名字增益小甚至近零——命名即检索 UX。

**泼冷水。** 最强仍 0.613；rubric 含 LLM judge，虽有三次 rescore 稳定性分析，Boundary/完成项噪声更大；任务是「单相关技能」设定，未覆盖多技能冲突与错误触发惩罚的完整生态。

## 🔬 最有意思的部分

1. **门控 SU：没 Trigger 就零执行分。** 防止「碰巧做对 / 没用技能」刷榜——这是对技能生态最狠也最必要的一刀。
2. **双瓶颈独立。** 会触发却不合规，与合规潜力高却不触发，是两类病；修技能文档 vs 修 harness 检索 UX 对症不同。
3. **Harness 条件化。** CC 利好「会找技能」的模型；Codex 抬触发、却更榨「进门后的持续照做」——和 [Harness-Design](../202609/week3/20260918_Harness-Design_论文解读_脚手架不是一整块_组件要按模型和预算配.md)「组件按模型配」同构。
4. **任务结果故意踢出 SU。** 避免「结果对了就当技能会了」——过程合同与交付物解耦。
5. **构造管线本身可复用。** masked-skill + 对抗审查 + 轨迹回放，是做 skills 评测基础设施的样本，不只是又一张表。

## 🤔 我的判断

定位：**progressive disclosure 下的技能使用过程基准 + CC/Codex 双 harness 实证**（测量基础设施，附失败分型）。亮点三：

1. Trigger/Compliance/Boundary + 门控 SU，把「会不会用技能」可运营化；
2. 真实技能 + Docker + 轨迹 rubric，贴近生产 coding-agent；
3. 明确 skill use 是 harness-conditioned——打脸「只换模型不换脚手架」的排行榜幻觉。

局限：单技能设定；SU 距「可靠」尚远；judge 项需持续标定。对 Byron：**(1)** 星标技能库前先问能否过 Skill-Use 式门控——否则只是收藏夹；**(2)** MMP / Pi 的 Skills 暴露面应区分「目录元数据」与「全文」，并打点 Trigger 率，而不是只看任务绿；**(3)** 双跑 CC vs Codex（或 ACP 多 harness）再谈模型排名，接 [Raven](./20261001_Raven_论文解读_别再手搓更强harness_Harness的Harness才把异构agent当可组合单元.md) 的异构编排；**(4)** 合规/边界项优先确定性检查，LLM judge 垫底——对齐 [LLM-as-Judge](../202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 与 [SkillSpec](../202609/week4/20260923_SkillSpec_论文解读_第一次用Hoare规格验Agent技能_515个近半有缺陷.md) 的规格验技能方向；**(5)** 与今日 [MetaSkill](./20261002_MetaSkill_论文解读_别把技能塞给Target_教Builder学元技能才配得动harness.md) 对照：那边学「如何为 Target 配支援」，这边测「Target 会不会用已配技能」——skills 闭环的上下游。

一句话收束：**挂上技能只是开始——Skill-Use 证明在 progressive disclosure 下，最强也只到 SU≈0.61，且换 Claude Code / Codex 排名会变：技能使用是 harness 条件化能力。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
