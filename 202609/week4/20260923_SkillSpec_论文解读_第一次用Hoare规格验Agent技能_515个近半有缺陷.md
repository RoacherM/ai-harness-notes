---
month: 202609
week: 4
date: 2026-09-23
type: 论文解读
slug: SkillSpec
---

# 第一次用 Hoare 规格验 Agent 技能：515 个里近一半有缺陷

你有没有过这种体验：skills 仓库越积越多——`SKILL.md` 写得像说明书，底下挂着 Py / JS / TS / Shell 脚本，跑起来「偶发沉默失败」，模型还把错执行圆过去，单元测试根本测不到「意图冲突」？北航 SkillSpec（arXiv:2609.06052）把问题抬到形式化：技能正确性不是「脚本能不能跑」，而是 **声明意图 vs 编码行为** 是否一致。他们做了据称**首个**面向 agent skill 的 Hoare 式规格推理框架：统一图（工作流 DAG + 语言无关代码 IR）上比 ExpectSpec 与 FactSpecs，再丢进沙箱验证。看完我的感受是：这正好卡在 [Code2Skill](../week3/20260921_Code2Skill_论文解读_还没跑轨迹也能攒技能库_从两万仓库榨出百万条.md)「怎么量产技能」和 [Subagents-vs-Skills](../week3/20260921_Subagents-vs-Skills_论文解读_技能该塞进主上下文还是开子代理.md)「技能怎么装载」之间缺的那一层——**技能质量保障（QA）**。

## 核心摘要

SkillSpec 将异构技能库变成统一图：从 `SKILL.md` 抽工作流 DAG，用 Tree-sitter 把 Py / JS / TS / Shell 解析成语言无关代码 IR，再用意图–实现双向绑定把节点对齐。对每个节点：从周围声明意图推 **ExpectSpec**；在 intent-mask（holistic / lineage / neighbors / self）下从编码行为推多视角 **FactSpecs**——遮太多会偏见「看起来该对」，遮太少会胡猜边界。多视角联合打出候选缺陷后，在隔离沙箱（OpenCode harness + Harbor 环境）自动验证。**515** 个真实技能（SkillsBench + skills.sh 高下载）上，标出 **239** 个有缺陷（**46.4%**），人工确认 **763** 条缺陷，整体精确率 **61.2%**；代码节点精确率 **67.1%**，工作流节点 **55.4%**。缺陷 taxonomy：requirement / constraint / conflict / implementation——工作流侧需求错误占比最高（约 43%），代码侧则以实现偏离为主（约 92%）。作者结论很刺：**纯文本工作流节点是瓶颈**；多数缺陷发生在「声明意图 ↔ 实现」边界。

## 论文信息

- **标题**：SkillSpec: Intent-Masked Specification Reasoning for Agent Skill Correctness
- **作者**：Yizhuo Zhang、Bo Kang、Yi Yang、Zhiyu Duan、Zhouteng Ye、Shunkun Yang
- **机构**：Beihang University（北京航空航天大学）
- **链接**：https://arxiv.org/abs/2609.06052 · 代码 https://github.com/IainZhang/SkillSpec · 数据集 https://huggingface.co/datasets/IainZhang/SkillSpec
- **观察时间**：2026-09-23

---

## 🎯 为什么这件事值得写

Agent 生态正把「可复用技能」当成经验与领域知识的打包单位：说明文字 + 异构资源。失败模式却超出传统代码缺陷——**意图冲突、约束遗漏、边界悄悄漂移**，表现成被底层模型掩盖的 silent failure。常规测试卡在可执行断言；LLM 直接「读一下 SKILL.md」又容易被全文意图带偏。

Byron 侧已有 [Code2Skill](../week3/20260921_Code2Skill_论文解读_还没跑轨迹也能攒技能库_从两万仓库榨出百万条.md)（从仓库榨技能）、[GraphSkillEvo](../week3/20260921_GraphSkillEvo_论文解读_技能别再写成一长串checklist_做成步骤图再进化.md)（步骤图进化）、[Subagents-vs-Skills](../week3/20260921_Subagents-vs-Skills_论文解读_技能该塞进主上下文还是开子代理.md)（装载位置）。SkillSpec 补的是：**技能进库之前 / 进库之后，如何系统验「说的和做的是不是一回事」**——对 skills/harness 工程几乎是刚需。

## 🏗️ 机制：Hoare 合同 + Intent-Mask

经典 Hoare 三元组 \(\{P\}C\{Q\}\) 在这里被改写为技能合同：

- **ExpectSpec**：节点周围声明意图定义的「应该怎样」；
- **FactSpecs**：在部分披露意图下，从工作流结构与代码 IR 推出的「实际怎样」；
- **正确性**：两套规格的行为一致性，而不是单看脚本 exit code。

统一表示三件套：

| 构件 | 来源 | 作用 |
| --- | --- | --- |
| 工作流 DAG | `SKILL.md` 半结构化说明 | 步骤、依赖、声明约束 |
| 代码 IR | Tree-sitter CST → 语言无关实体/调用 | 跨 Py/JS/TS/Shell 对齐 |
| 意图–实现绑定 | 双向节点链接 | 把「写的」钉到「跑的」 |

Intent-mask 四层可见性（holistic / lineage / neighbors / self）刻意折中：全盘意图会把 Fact 抽成「复读 Expect」；只看局部又会发明不存在的任务边界。候选缺陷必须过沙箱：OpenCode + Harbor，每技能复用暖容器、每候选独立工作区；验证模型用 Grok 4.6 以更好吃异构技能，单候选最多三轮。

## 🧪 主结果：近半技能带伤，代码侧更好验

- **覆盖面**：515 技能 → 239 有缺陷（46.4%），763 条人工确认缺陷；缺陷精确率 61.2%。
- **节点类型**：代码节点 67.1% precision vs 工作流 55.4%——规格推理在可结构化 IR 上更稳，**plain-text workflow 是短板**。
- **Taxonomy（人工确认）**：

| 类别 | 工作流占比 | 代码占比 |
| --- | --- | --- |
| Requirement Error | 43.1% | 1.7% |
| Constraint Omitted | 13.3% | 6.7% |
| Semantic Conflict | 17.6% | 0% |
| Implementation Deviation | 26.0% | 91.6% |

读法：工作流常「需求写错 / 约束漏写 / 语义打架」；代码几乎都是「实现偏离声明」。消融表明 Hoare 式规格与多视角 mask 都贡献精度；跨模型族时代码侧结论更一致。

## 🔬 最有意思的部分

1. **Silent failure 的形态化**：意图冲突可以在「技能表面上能跑完」时仍成立——这解释了为何只靠轨迹 reward / 单测会漏。
2. **Mask 不是抠细节**：过多上下文是偏见源，过少是幻觉源；四层视图是在给 LLM 规格推理做「信息暴露课程」，很像评测里的 anti-leakage。
3. **瓶颈在散文式工作流**：越是自然语言步骤清单，规格越难钉死——和 [GraphSkillEvo](../week3/20260921_GraphSkillEvo_论文解读_技能别再写成一长串checklist_做成步骤图再进化.md)「别再写成一长串 checklist」同向：结构化步骤图不仅利于进化，也利于 QA。
4. **工程闭环**：静态规格候选 + 沙箱动态验证，比「再让一个 agent 审一遍」更可审计；开源仓库与 HF 数据集降低复现门槛。

## 🤔 我的判断

定位：**把形式化规格推理搬进 agent skill QA 的测量 + 方法论文**，同时贡献真实缺陷分布。亮点三：

1. 问题定义准——技能失败是意图–实现一致性，不是「脚本 lint」；
2. 统一图 + intent-mask 把异构工件变成可推理对象；
3. 46.4% 带缺陷的基线数字，足够让任何 skills 仓库维护者紧张。

局限：61.2% precision 仍有假阳性成本；工作流侧更弱；沙箱验证依赖另一套 agent/模型，可能引入验证偏见；外部效度绑定 SkillsBench / skills.sh 生态。对 Byron（skills / harness / Wayne-Skills）：**入库门禁至少要有「声明意图 vs 实现」检查**——Code2Skill 式量产之后若不做 SkillSpec 式 QA，等于在主上下文里塞未经验证的程序。和 Subagents-vs-Skills 拼：那边争论装载位置，这边提醒——**有缺陷的技能，装哪都危险**。

一句话收束：技能不是文档附件，是带隐含 Hoare 合同的可执行工件；**先对齐 Expect 与 Fact，再谈复用与进化**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
