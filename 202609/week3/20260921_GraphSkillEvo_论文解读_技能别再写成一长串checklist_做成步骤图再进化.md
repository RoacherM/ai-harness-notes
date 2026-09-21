---
month: 202609
week: 3
date: 2026-09-21
type: 论文解读
slug: GraphSkillEvo
---

# 技能别再写成一长串 checklist：GraphSkillEvo 把流程做成可进化的步骤图

你有没有过这种体验：给 agent 塞一份「优化过」的 skill，结果还是又长又散的 bullet——弱模型跟丢步骤，强模型也在冗余表述里打转；而优化器本身（如纯 LLM 自反思的 SkillOpt）只能在无结构自然语言空间里打补丁，搜索又贵又容易早停。CityU / NUS / 南科大这篇 GraphSkillEvo（arXiv:2609.21749；亦见 2026-09-21 HF Daily Papers）把 skill 重写成**图结构自然语言制品**：节点 = 步骤 + 操作指导，边 = 情境依赖的转移路径，再用种群级突变与交叉在结构化技能空间里搜。看完我的感受是：它把「技能表示」和「技能搜索」绑成一件事——对 Byron 这种 skills / harness 取向，这是 [EvoSkill-GUI](./20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) 式进化之外，另一条**显式流程先验 + 种群重组**的路线。

## 核心摘要

GraphSkillEvo 形式化 \(s=\langle h_s,g_s\rangle\)：全局指导 \(h_s\) 放跨步骤原则；有向图 \(g_s=(V_s,E_s)\) 里每个节点是可复用执行步骤（含该步规则与约束），边通过 \(M\) 条带适用条件的 workflow 路径定义，不同情境可共享节点但走不同路径。优化侧维护种群大小 \(N=4\)、跑 \(T=5\) 代；每代在训练批上滚出轨迹，用至多 \(K=5\) 条失败轨迹做反思，按轮询调度四种算子（全局指导突变 / 图结构突变 / 全局指导交叉 / 图结构交叉），再按验证集适应度保留 Top-\(N\)。主表（三跑平均）在五套 agent 基准（SearchQA、SpreadsheetBench、DocVQA、LiveMath、ALFWorld）上：无 harness 时相对 SkillOpt，GPT-5.4-nano 均分 **+4.01%**（56.89→60.90）、GPT-5.4 **+1.76%**（74.10→75.86）；挂 Codex harness 的 GPT-5.4 再 **+1.33%**（76.95→78.28）。14 个模型–harness–基准设定里 **13** 个最优；Spreadsheet 上相对 SkillOpt 可到 **+10.60**（nano：50.11→60.71）。消融：去掉图结构平均 **−7.44**，去突变 **−17.02**，去交叉 **−4.93**；仅保留节点文案、去掉显式 workflow 的「非结构化对照」在五基准上亦全面掉点。优化 token 总量：SkillOpt 约为 GraphSkillEvo 的 **1.31× / 1.36×**（GPT-5.4 / nano）。nano 上优化出的图技能还能迁到 GPT-5.4，Spreadsheet 上 transferred **71.78** 甚至超过 direct **69.40**。

## 论文信息

- **标题**：GraphSkillEvo: Evolutionary Optimization of Graph-Structured Agent Skills
- **作者**：Rui Sun*、Zhi Zheng*、Zhenkun Wang、Zhichao Lu（* 共同一作）
- **机构**：City University of Hong Kong · National University of Singapore · Southern University of Science and Technology
- **链接**：https://arxiv.org/abs/2609.21749 （v1，2026-09-18）· 代码 https://github.com/ruisun7/GraphSkillEvo

---

## 🎯 无结构技能的两道坎：执行跟丢 + 搜索空间爆炸

论文把现有 skill 优化（以 SkillOpt 为代表）钉在同一表示病根上：

1. **执行难**：优化后的 skill 常变成冗长 checklist，只给粗粒度流程；agent（尤其 GPT-5.4-nano 一类较弱模型）难以判断「当前该看哪条、下一步是什么」。
2. **优化难**：无结构自然语言可对同一流程写出无数字面变体，优化器大量时间耗在**无实质程序差异**的文本改写上。

动机很直观：技能本就该像流程图——步骤自带操作说明，转移随情境分支。把图结构写进 skill 制品本身，既给执行端 workflow 先验，又给优化端更紧凑的搜索空间；再叠种群交叉，比「单技能反复 patch」更能跨轨迹拼装有效组件。这和 [Code2Skill](./20260921_Code2Skill_论文解读_还没跑轨迹也能攒技能库_从两万仓库榨出百万条.md) 互补：Code2Skill 解决**技能内容从哪来、如何验**；GraphSkillEvo 解决**已有技能如何结构化再搜**。与 [Subagents-vs-Skills](./20260921_Subagents-vs-Skills_论文解读_技能该塞进主上下文还是开子代理.md) 连读：图里的节点/路径，天然更适合被当成契约化包的「内部日程」。

## 🏗️ 机制：全局指导 + 节点步骤图 + 四种结构感知算子

**表示。** 示例（家庭 ALFWorld 风格）：全局原则（勿连续重复同一动作超过两次、卡住就换未探索位置、只从 admissible 列表选动作……）；节点如 Explore Object / Take Object；边通过「Pick & Place」等 workflow 声明 Use When + 有序步骤序列。共享节点只写一次，多条路径引用——低冗余。

**进化回路（四步循环）。**

0. **初始化**：种群 \(P^{(1)}=\{s_i\}_{i=1}^{N}\)，除种子 skill 外用 LLM 生成多样图技能，全量 \(D_\mathrm{val}\) 打适应度。
1. **训练执行**：每代从 \(D_\mathrm{train}\) 采样 \(B=15\) 实例，每技能出轨迹，保留失败轨迹作突变反思（\(\lvert F_i\rvert\le K=5\)）。
2. **生成子代**：轮询选算子；父代按 \(p\propto 1/(r+N)\)（\(r\) 为适应度名次）抽样；突变带失败轨迹，交叉只重组结构/指导。
3. **选择**：新 \(N\) 个候选与旧种群合并，按 \(J_{D_\mathrm{val}}\) 留 Top-\(N\)，共 \(T=5\) 代后取最优。

四种算子（Figure 2）：**全局指导突变**改原则、保图；**图结构突变**改节点文案、增删节点、调路径，保全局指导；**全局指导交叉**拼两份指导、保一方图；**图结构交叉**拼节点/边、保一方指导。目标始终是固定 LLM 参数与 harness，只搜 skill 制品 \(s^\star\in\arg\max_s J_D(s)\)。

评测覆盖无 harness（skill 写入指令）与 Codex harness（SDK、workspace-write、读 skill）；ALFWorld 因需持久环境交互，Codex 格留空。执行与优化用同一 LLM（GPT-5.4 / GPT-5.4-nano，medium reasoning）。

## 🧪 关键证据：弱模型与流程型任务吃到最大红利

### 主结果（Table 1，三跑平均）

相对 SkillOpt 的平均增益叙事已在摘要；再抓几处工程敏感点：

| 设定 | 相对 SkillOpt |
| --- | --- |
| GPT-5.4-nano / 无 harness / 五基准均 | **+4.01**（56.89→60.90） |
| GPT-5.4 / 无 harness / 五基准均 | **+1.76**（74.10→75.86） |
| GPT-5.4 / Codex / 四基准均（无 ALFWorld） | **+1.33**（76.95→78.28） |
| nano · Spreadsheet | **+10.60**（50.11→60.71） |
| nano · ALFWorld | **+3.73**（57.46→61.19） |
| 唯一落后 | LiveMath · nano：GraphSkillEvo 28.76 vs SkillOpt 29.56（**−0.80**） |

相对无 skill：GPT-5.4 / nano / Codex 三路上平均提升约 **+15.37 / +21.86 / +10.31**。弱模型与程序化交互任务（表格操作、具身）增益最大——正是「显式 workflow」该发力的地方。优化曲线（Figure 3）：SkillOpt 较早平台，GraphSkillEvo 随 token 持续抬验证分。总优化 token（Table 2）：GPT-5.4 上 SkillOpt **81.08M** vs GraphSkillEvo **61.94M**；nano 上 **103.54M** vs **75.94M**。

### 消融与执行侧图结构（Table 3–4）

- 把优化好的图技能改成「保留全局+节点文案、去掉显式 workflow」：五基准分别 **−4.52 / −2.50 / −4.19 / −1.35 / −0.75**——执行端也需要路径组织，不只是多写几条节点说明。
- 优化消融（SearchQA+Spreadsheet+DocVQA 平均）：完整 **71.52**；无图结构 **64.08（−7.44）**；无突变 **54.50（−17.02）**；无交叉 **66.59（−4.93）**。无交叉 ≈ 多条并行 SkillOpt 式自精炼，说明交叉带来跨候选组件拼装；无突变则失去轨迹驱动的局部修正，掉得最狠。

### 跨模型迁移（Table 5）

nano 优化 → GPT-5.4 部署：三基准 transferred 均高于无 skill；Spreadsheet 上 GraphSkillEvo transferred **71.78** > direct **69.40**，且远高于 SkillOpt transferred **53.21**。技能作为可移植程序知识（而非绑死某一模型参数）的主张，这里给了定量支撑。

## 🔬 最有意思的部分：表示即搜索空间压缩

这篇最值得抠的不是「又一个进化算法」，而是**把自然语言 skill 的自由度钉死在「节点+条件路径」坐标系里**——优化器改的是程序组件，而不是同义改写。轮询四种算子 + 小种群（\(N=4\)）就能在更少 token 下压过单轨迹自反思，说明瓶颈常在表示，不在「再多打几轮 patch」。和 [EvoHarnessBench](./20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) 对读：一边在技能空间进化，一边要防 harness / 技能库膨胀带来的 forgetting——图技能若持续增殖，仍需 engagement / BWT 式监测，否则「搜得动」会变成「库失控」。与 [Meta-Harness](./20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 的差异也清晰：Meta-Harness 搜的是脚手架与反馈形态；GraphSkillEvo 搜的是**固定 harness 下的 skill 图**——两者可叠，但层级不同。

## 🤔 我的判断

定位：**结构化技能表示 + 种群进化搜索**的方法论文，直接服务 agent skill 优化栈。亮点三：

1. 把「难执行」与「难优化」归因到同一无结构表示，并用图节点/条件路径一次性对症；
2. 突变管轨迹局部修正、交叉管跨候选拼装，消融数字清楚，不是口号式 EC；
3. 弱模型与流程型基准增益大、优化 token 更省、跨模型可迁移——工程可搬点明确。

局限与代价：主设定 \(N=4,T=5\)、每代仅 15 训练实例，搜索深度有限；LiveMath·nano 仍略逊 SkillOpt；Codex 路径未覆盖 ALFWorld；未来工作自己点出需与参数化优化结合、更丰富图组合、跨域图技能合并。对 Byron：Wayne-Skills / harness-dojo 若已有非结构化 skill 库，可把「步骤+适用条件」升成图，再对高价值任务跑小种群进化；灌库用 Code2Skill，表示与进化用 GraphSkillEvo，执行模态按 Subagents 文选——**挖得到、画得清、搜得动、迁得走**，才是完整技能供应链。

一句话收束：checklist 是给人类扫一眼的，agent 更需要一张能执行、能突变、能交叉的步骤图——GraphSkillEvo 把这张图变成可优化的搜索对象。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
