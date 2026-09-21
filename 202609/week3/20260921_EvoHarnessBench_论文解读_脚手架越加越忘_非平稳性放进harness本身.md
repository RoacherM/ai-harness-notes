---
month: 202609
week: 3
date: 2026-09-21
type: 论文解读
slug: EvoHarnessBench
---

# 脚手架越加越「忘」：EvoHarnessBench 把非平稳性放进 harness 本身

你有没有过这种体验：给 coding agent 多挂一批 MCP / skills / 子代理，以为能力只会涨——结果老任务突然不稳、token 飙升、delegation 漂移，模型权重一行没动。行业里「持续学习」评测大多把非平稳性放在**任务流**上，harness（工具、技能、专家代理）却当静态背景。Salesforce Research / UNC / Wisconsin 这篇 EvoHarnessBench（arXiv:2609.04280v2）反过来问：**外供 harness 自己在涨时，agent 还能不能跟上？** 看完我的感受是：它不是又一个 agent 榜，而是把「脚手架膨胀」独立成一等公民的测量基础设施——和本站已归档的 [Meta-Harness](./20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 形成闭环：那边教你怎么搜 harness，这边量你加完之后会不会「忘」。

## 核心摘要

EvoHarnessBench 沿 **tools / skills / agents** 三轴构造 **17** 条多阶段 harness 流（每条 **3–6** 阶段），由 verifier 基准确定性嵌套：累计 **802** 任务、**1,510** 条轴向评测例、**520** 可执行工具、**42** 参考技能、**62** 专家代理。相对既有 continual-agent 基准，它把非平稳性放在**外部供给的 harness**，而不是任务序列。两种评测模式：**deployment**（无持久状态、各阶段独立）隔离「只因 catalog 变大」的效应；**self-evolving adaptation**（记忆 / prompt / 代码级适配，权重固定）测早期经验在新能力涌入后是否仍有用。三大缺口：(1) **harness-induced forgetting**——权重未变，仅因 harness 扩张，旧能力可丢，Figure 1 部署端工具/技能/代理分别掉 **12.1% / 13.8% / 46.4%**，agents 轴 ALE 上部署 BWT 最差达 **−34.7%**；(2) 自进化收益跨环境不一致——EOG 上 MemToolAgent / Meta-Harness 等可抬升，ALE 上多数贴近或低于部署基线；(3) **保留 vs 适应张力**——BWT↑ 常伴随 FWT↓（如 tools 轴 ALE 上 GEPA / Meta-Harness 的 FWT 到 **−28.5% / −11.1%**）。跨轴对照表：工具扩张可提准确率但抬 token；技能池部署端几乎「免费」却难 engagement（默认 GPT-5 几乎不调用）；代理最难——Meta-Harness 把 EOG agents 从 **8.8%→18.5%**（相对 **+110.2%**）。结论动机明确：需要把外环 harness 演化与内环适配**双层耦合**，而不是孤立优化记忆。

## 论文信息

- **标题**：EvoHarnessBench: Can Your Agents Keep Pace with an Evolving Harness?
- **作者**：Zixuan Ke∗†、Vaidehi Patil∗†、Haizhou Shi∗†、Yang Li†、Ye Liu、Sarath Shekkizhar、Anurag Koul、Jiayu Wang、Xuan Phi Nguyen、Semih Yavuz‡、Mohit Bansal‡、Shafiq Joty‡（∗ 共同一作；† 核心；‡ 资深）
- **机构**：Salesforce Research · UNC Chapel Hill · University of Wisconsin–Madison
- **链接**：https://arxiv.org/abs/2609.04280 （v2，2026-09-10）· 项目 https://mas-orchestra.salesforceresearch.ai/evoharness/

---

## 🎯 为什么「任务非平稳」不够用

真实 Agentforce / openai/skills 类生态在持续加减能力（论文用 Figure 2 绿线对照真实增长）。若评测只换题、不换脚手架，你会漏掉一整类失败：**所需工具仍在池子里，但被更大的 alternatives / 协调噪声淹没**。作者把这叫 harness-induced forgetting——和权重侧 catastrophic forgetting 同形异因：参数不动，**行动界面变了**。

两难拆开写很干净：

1. **Retention**：旧任务所需能力还在，但嵌在更大 pool 里，原先可靠行为难恢复；
2. **Adaptation under accumulated experience**：在窄 harness 下攒的 memory / skill / prompt / routing，在宽 harness 下可能过时甚至误导。

对照 Table 1：静态 harness 榜、任务流 continual 榜、自生长能力榜都没有「外供 harness 演化」这一维。EvoHarnessBench 填的是测量缺口，不是又一个 SOTA 方法。

## 🏗️ 基准机制：外环演化 × 可选内环适配

**外环**：沿单轴（tools / skills / agents）构造嵌套累积流 \(H_1 \subset H_2 \subset \cdots \subset H_T\)；能力按「高频核心 → 长尾」释放；每个任务挂到「所需能力首次齐备」的最早阶段，且至少用到该阶段新引入的一项。另设 **task-specific reference**：每题只暴露标注必需能力，用来对照「窄曝光 vs 累积目录」。

**内环（可选）**：阶段 \(t\) 可用累积 adapt split 更新持久态 \(z_t\)（权重固定）。部署模式关掉内环；自进化模式打开，并在 held-out eval 上测。

**数据来源**：EOG（有状态企业工作流 + 结构化工具标注）与 ALE（多样 agentic + 富软件工具环境）拼成 17 流。指标除 pass / score / 时间 / token 外，显式报 **FWT**（对新阶段 cohort 的适应）与相对 **BWT**（harness 扩张后对更早 cohort 的表现变化）。

适配族覆盖三类，便于和 Byron 栈对照：

| 族 | 代表 | 改什么 |
| --- | --- | --- |
| Memory | Raw Memory、ReasoningBank、MemToolAgent、G-Memory、LEGOMem | 情节经验存取 |
| Prompt | GEPA | 持久文本指令 |
| Code | **Meta-Harness** | 可执行 harness 本身 |

## 🧪 三轴主结果：工具涨、技能闲、代理崩

### Tools：准确率可涨，代价与遗忘并存

相对 task-specific，全量工具目录让 frontier 部署 pass 从 EOG **26.0%→30.2%**、总体 **24.2%→28.1%**，但 token 从 **27.2M→75.8M**（Obs.❶）。自进化在 EOG 上 MemToolAgent **38.6%**、ReasoningBank **36.9%**、Meta-Harness **35.2%** vs 无适配 **30.2%**；ALE 上多数贴住或低于部署基线（Obs.❷）。Figure 3：无适配时 BWT 已负；GEPA / Meta-Harness 可把 BWT 拉正，但 ALE 上 FWT 崩到 **−28.5% / −11.1%**——**保旧伤新**（Obs.❸）。适配期若收窄到 task-specific 工具，MemToolAgent 最终 pass 几乎不变（**38.6→39.1**），适配时间 **23.9h→17.9h**——难在「宽工具空间里学」，不只在「分阶段演化」。

### Skills：部署几乎无感，engagement 才是瓶颈

Codex 在 EOG 上 task-specific 与累积技能池同为 **18.9%** pass，token 几乎不动（Obs.❶）——技能要显式检索调用，不像工具那样持续占动作空间。GEPA 把 EOG 提到 **24.1%**，其余方法更散（Obs.❷）。Figure 5 更刺眼：默认 GPT-5 在终阶段对 offered skills 调用率接近 **0%**；task-specific GPT-5.5 到 **82%**，Claude Code **15%**；task-specific GEPA 可把 GPT-5 拉到 **31%**。技能学习消融还显示：顺序跨阶段攒库不如一次性联合学同一批经验（Batch Teacher Feedback **22.1% vs 23.6%**）。这和 [EvoSkill-GUI](./20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) 的提醒同向：**库在不代表用得上**。

### Agents：最难轴，Meta-Harness 相对涨幅最大

即使 task-specific 参考池，EOG Codex 也只有 **6.5%**；扩到累积池 **8.8%**，ALE 则 **5.3%→4.2%**——环境依赖极强。自进化在 EOG：Meta-Harness **18.5%**（相对部署 **8.8%** 即 Table 8 的 **+110.2%**），G-Memory **18.2%**，GEPA **14.9%**；ALE 上多数 ≤ **4.2%**，仅 GEPA 明显改善。Figure 6：ALE 部署 Codex **BWT −34.7%**——三轴最强遗忘。机制分析：遗忘任务的 delegation drift（**13%→25%**）更大，主因是**丢掉曾成功的 required-agent 覆盖**，而非简单被新 agent  distract；适配主要抬高 required-agent recall（约 **79%→89%**），selection precision 本来就在 ~90%。

Table 8 跨轴一览（论文原文）：

| | Tools | Skills | Agents |
| --- | --- | --- | --- |
| 部署扩张 | Accuracy↑, cost↑ | 几乎无感 | 环境依赖 |
| 最大 harness 遗忘 | **−5.3%** | **−4.0%** | **−34.7%** |
| 最佳适配增益 | +27.8% MemToolAgent | +27.5% GEPA | **+110.2% Meta-Harness** |
| 瓶颈 | 从宽工具空间抽有用经验 | 调用对的技能 | 覆盖 vs 选择性 |

（Figure 1 的 **12.1 / 13.8 / 46.4%** 是另一套部署掉点汇总；与 Table 8 的 BWT 峰值并列引用时注意口径：前者偏整体 drop 叙事，后者是相对 BWT。）

## 🔬 最有意思的部分：双层适配与「保旧伤新」

结论段直接开药方：**bi-level adaptation**——内环用当前 adapt 经验更新 \(z_t\)，外环目标检验这些更新是否在后续 harness 阶段仍有效，在「吃新能力」与「保旧能力」之间显式权衡。另需检测何时持久产物已 stale / 有害，并选择性修订或丢弃；未来还应覆盖能力**替换与退役**，而不只是单调增长。

对做 judge / eval 的人：聚合终局 pass 会掩盖轨迹上的 FWT/BWT 撕裂——这和 [LLM-as-Judge](./20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 强调的「分层门禁 + 切片指标」同构：你至少要沿 harness 阶段切片，而不是只报最后一个 catalog 的总分。

## 🤔 我的判断

定位：**harness 演化的测量基础设施 + 问题定义论文**，方法贡献是把「外供脚手架非平稳」从任务流 continual learning 里剥离出来。亮点三：

1. 三轴 × 双模式设计干净，802/520/42/62 规模够用，FWT/BWT 强迫你看见保留–适应张力；
2. 把 Meta-Harness / GEPA / MemToolAgent 放进同一外环演化协议，直接回答「搜 harness / 改 prompt / 攒记忆」谁在哪轴有效；
3. agents 轴的 delegation drift 解释比「池子太大分心」更深——是**成功覆盖被冲掉**。

局限：Claude Code 因模型/harness 不同未进主表；部分 MAS 预算内未跑完；skills 轴依赖「latent reference skills」与合成 engagement 叙事；真实生产还有删除/重命名/权限变更，论文承认单调增长只是第一步。对 Byron 栈：Sep 18 已归档 Meta-Harness——这篇是它的**压力测试说明书**：脚手架可搜、可进化，但若不做外环–内环双层目标，你可能用完整轨迹把 harness 改得更「适应当前阶段」，同时在 agents 轴上制造 −34.7% 级遗忘。和 ComposeCL（权重侧组合）、EvoSkill-GUI（部署时技能自进化）并列时，分工是：**权重 / 技能内容 / 脚手架目录**三条非平稳线要分开测、再耦合治。

一句话收束：加 MCP、加 skill、加 subagent 之前，先问一句——**你的评测有没有沿着 harness 版本量过 BWT？** 没有的话，榜上的涨可能只是在透支旧任务。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
