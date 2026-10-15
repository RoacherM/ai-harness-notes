---
month: 202610
week: 3
date: 2026-10-15
type: 论文解读
slug: WorkForge
---

# 可验证工作环境也能规模化：真实工作区事实锚定任务与裁判

你有没有过这种体验：想训「真干活」的 work agent——审项目、对表、写交付物——可手搓环境工程量炸裂；一改自动合成，工作区又变成干净小树，证据全挤在两三个显眼文件里，裁判还容易编出工作区里根本没有的要求。**真实摩擦和可验证监督，好像总要二选一。**

腾讯 LLM 部与复旦的 Jiazheng Zhang、Long Ma、Yunxian Yang、Zhiheng Xi、Tao Gui、Qi Zhang 等这篇 **WorkForge**（arXiv:2610.04906）的回答是：先从专家工作流抽出 Scene Blueprint，再检索真实资源落成工作区，**从可检查的工作区事实（factual anchors）反向生成任务说明、隐藏解法计划，以及程序化+语义双裁判**——让问什么、验什么都钉在可观察证据上。看完我的感受：这是把「环境合成」从玩具仓库拉回职业交付物考场，而且缩放曲线干净——数据量与交互地平线都涨分。

## 核心摘要

WorkForge 四段流水线：(1) 专家工作流 → Scene Blueprint（工作流/交付物/资源依赖/决策点）；(2) 采集 agent 检索真实资源实例化工作区并做 profiling（文件角色、依赖、事实锚、artifact schema）；(3) 由蓝图+画像联合生成可见指令与隐藏计划，再从事实锚导出程序化检查与 LLM 语义 rubric；(4) solver 在可见工作区上 rollout，双裁判打分，沉淀轨迹。语料：**16,665** 个可验证环境，跨 **40** 个专业域，平均 **42.3** 文件 / **279 KB** / **60** 种文件类型，平均轨迹 **31.8** 步、**33.8** 次工具调用。SFT Qwen3.5-35B-A3B-Base：GDPval **45.5→73.6**（+28.1），APEX **5.0→21.3**（+16.3），WildClaw / ClawEval / ALE 同步抬升；Qwen3.5-27B 再训后 GDPval **82.4**，超过近 15× 参数的 Qwen3.5-397B-A17B（79.7）。消融显示 program+semantic 混合选轨迹（Hybrid ALE 22.11）显著优于单路；真实工作区相对合成约 **5.4×** 文件、**2.4×** 类型、体积 **21.5→279 KB**。价值定位：**可缩放的、事实锚定的 work-agent 环境基础设施**。

## 论文信息

- **标题**：Scaling Verifiable Environments for Long-horizon Work Agents
- **作者**：Jiazheng Zhang, Long Ma, Yunxian Yang, Zhiheng Xi, Zhikai Lei, …, Tao Gui, Qi Zhang, Xuanjing Huang, Clarenceai, Aiden Adams
- **机构**：Tencent LLM Department · Fudan University
- **链接**：https://arxiv.org/abs/2610.04906 （v3 2026-10-13；HF Daily Papers 约 2026-10-14/15）

---

## 🎯 为什么这件事值得写

Work agent 的考场不该是「会不会调 API」，而是「会不会在嘈杂文件系统里找证据、做决策、交可验收交付物」。这和 [ARBITER](./20261015_ARBITER_论文解读_官方判对却不说错哪_可验证反例当代码裁判证据.md)「可验证反例当证据」、[AgentSWE](../week2/20261014_AgentSWE_论文解读_coding_agent交出来看着齐其实空_冻结提交加holdout_rubric才测得出真交付.md)「交付要 holdout rubric」同一证据观，也和 [LLM-as-Judge](../../202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 的顺序一致：**程序化可检查项先钉死，开放交付再用事实锚定的语义裁判。**

手搓环境保真但不可扩；纯合成环境便宜但探索需求被掏空（Figure 1）。WorkForge 的赌注是：**任务与验证都从同一份工作区事实长出来**，幻觉要求会少很多。

| 维度 | 合成工作区 | 真实资源工作区 |
| --- | --- | --- |
| 平均体积 | 21.5 KB | 279 KB（约 13×） |
| 文件数 | 7.9 | 42.3（约 5.4×） |
| 文件类型 | 25 | 60（约 2.4×） |
| 证据分布 | 集中、好猜 | 分散、要探索 |

## 🏗️ 机制：Blueprint → 真实工作区 → 事实锚 → 双裁判

**Scene Blueprint** $b_j=(W_j,O_j,R_j,D_j)$：工作流上下文、交付物、资源依赖、领域决策。例如「对照标准答案给学生作业分组讲解」可复用不同教材与答卷。

**Environment Mapping & Profiling。** 采集 agent 搜索/下载/预览资源池，实例化 $W_{jk}=\phi_{R_{jk}}(b_j)$；画像 $P_{jk}=(U_{jk},F_{jk},A_{jk})$ 含领域摘要、**事实锚 $F$**、artifact schema——决定「能问什么、能验什么」。

**Task & Verifier Design。** 可见指令 $q$ 与隐藏计划 $h$ 联合生成；计划中的关键声明回查工作区得证据 $\hat F$。程序化分数：
$$S^{\mathrm{prog}}=\frac{1}{|V^{\mathrm{prog}}|}\sum_{v\in V^{\mathrm{prog}}}\mathbf{1}[v(W,Y)=1]$$
语义分数由准则专属打分指南的 LLM judge 归一到 $[0,1]$。任务包显隐分离：agent 只见 $(W,q)$，计划与裁判隐藏。

**Trajectory Generation。** Solver 交互至提交或达上限；保留 $(\tau,Y,S^{\mathrm{prog}},S^{\mathrm{sem}},\mu)$ 供 SFT。主实验五 epoch、全局 batch 64、最长序列 **256K**、lr $1\times10^{-5}$。

## 🧪 关键证据

**主结果（Table 2，节选）。** WorkForge-A3B：WildClaw Score 18.6→**29.9**，ClawEval 35.4→**53.3**，GDPval **45.5→73.6**，APEX **5.0→21.3**，ALE 11.4→**28.7**（Pass@1 3.8→**12.4**）。在 <100B 专用基线中 GDPval 大幅领先 ClawGym-30A3（35.0）。WorkForge-27B：WildClaw +8.0、APEX +4.2、GDPval +3.0 至 **82.4**，逼近 Claude Opus 4.8（89.1）/ GPT-5.6 Sol（90.1）区间下沿。

**数据量缩放（Figure 4a）。** 同配方 1K→3.6K→8K→16K 轨迹，ALE / WildClaw 单调上升，16K 达 ALE **28.7**、WildClaw **29.9**。

**交互地平线（Figure 4b）。** 更长 turn 组监督带来更高分；附录还显示更长轨迹监督改变工具使用行为。

**Harness 对齐。** 各 8k 条 Claude Code vs OpenClaw 轨迹：在 Claude Code 评测下前者 ALE 更高（24.2 vs 16.1）；换 OpenClaw harness 则后者 WildClaw 高 **6.2**——存在 harness 特异迁移，训练与评测脚手架对齐有帮助。

**验证消融（Table 3，各 1K）。** LLM-only ALE 15.62 / APEX 11.18；Programmatic-only 14.89 / 10.92；**Hybrid 22.11 / 15.39**——双信号混合选轨迹明显更强。

**泼冷水。** 语义裁判仍是 LLM；蓝图依赖专家蒸馏；harness 特异迁移说明「环境通用」有限；主增益来自 SFT 轨迹质量，非在线 RL；公开复现细节需看附录与后续发布。

## 🔬 最有意思的部分

1. **事实锚同时约束出题与判分。** 不是先写开放题再找裁判，而是「工作区能支持什么事实 → 才能问什么」——从根上压幻觉要求。
2. **真实噪声是特性不是 bug。** 更大、更杂、证据更散，才逼出探索与跨文件整合；合成干净树测不出这能力。
3. **Hybrid 验证不是和稀泥。** 程序化钉死可观察量，语义评开放交付，1K 消融里混合显著优于任一单路。
4. **小 MoE 吃 WorkForge 可打过更大稠密/MoE 基线。** 27B 训后 GDPval 超 397B-A17B，说明监督形态比裸参数量更关键。
5. **和同日三篇形成控制栈拼图。** ActWrong 诊决策、FailConf 做结局置信闸、WorkForge 供可验证长程考场——环境、测量、控制三条线。

## 🤔 我的判断

定位：**可扩展的 work-agent 环境与轨迹基础设施**（测量+训练数据），不是单点算法花样。

亮点三条：

1. 真实资源工作区 + 事实锚闭环，同时解决「假环境」与「假裁判要求」；
2. 16.7K / 40 域 / 60 文件类型的规模，外加清晰的数据量与地平线缩放；
3. 程序化+语义混合选轨与主结果增益相互印证。

问题与局限：专家蓝图成本；语义裁判偏差；harness 耦合；长序列 SFT 算力门槛；伦理上真实第三方文件需清洗与授权。

对工程的直接启发：**(1)** 合成 work 环境时先问「事实锚从哪来」，没有可观察锚就不要生成开放 rubric；**(2)** 轨迹筛选默认 hybrid，别只信 LLM judge 分；**(3)** 训练与部署 harness（工具约定、turn 预算）尽量对齐；**(4)** 评 work agent 要看跨文件整合与交付物 rubric，而不是单文件问答；**(5)** 可与 [ARBITER](./20261015_ARBITER_论文解读_官方判对却不说错哪_可验证反例当代码裁判证据.md) 思路叠用——程序化反例/检查做硬闸，语义分做软分。

一句话收束：**能规模化的不是「更多假仓库」，而是「从真实工作区长出来的、事实可追溯的任务与双裁判」。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
