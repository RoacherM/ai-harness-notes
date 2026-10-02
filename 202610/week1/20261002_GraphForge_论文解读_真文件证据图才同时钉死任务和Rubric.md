---
month: 202610
week: 1
date: 2026-10-02
type: 论文解读
slug: GraphForge
---

# 真文件 + 证据图，才同时钉死任务陈述和可验证 Rubric

你有没有过这种体验：给 working agent 造训练轨迹时，要么让模型**编文件**（看着像样、一查就假），要么趴在真实 PDF/XLSX 上出题却**没有任务专属 verifier**——交付物「看起来专业」，分数全靠心情。中科大 / 复旦 / 上海创智 / 上海 AI Lab 这篇 **GraphForge**（Training Working Agents with Graph-Anchored Workspace Synthesis，arXiv:2609.38923）把种子与文件角色拆开：O*NET 职业种子控多样性，真实文件搭 workspace，再在文件关系上建 **evidence graph**——任务陈述与评分准则都从这张图编译出来，每条 criterion **锚到核验所需的源文件**。看完我的感受：这是 Byron **rubric / LLM-as-judge · 工作区 agent · OpenClaw 式 working agent** 一条极硬的数据基建——**2,169 条高分轨迹**把 Qwen3.6-27B 在 OpenHands 上的 GDPVal 推到 **1445.7（+65.7 Elo）**，且增益跨 Codex / Claude Code 脚手架转移。

## 核心摘要

Working agent（读多文件、调工具、交可交付物）的 SFT 需要「真文件 + 可验证结果」，但 EnvCraft 类管线合成文件、状态脚本读不了文档内容；NexForge 类用真文件却无任务级 rubric。GraphForge 五段漏斗：职业种子 → 覆盖式检索真实文件组成 workspace → 模型编 evidence graph → 编译任务与**证据锚定 rubrics** → 教师一步 rollout 测可执行性，revision agent 只修任务/准则（对照原文件）→ 制品级确定性检查 + 证据锚定 judge 准入（\(Q>0.90\)）。SFT 语料 2,169 轨，覆盖 466 类 O*NET 任务、15/16 职业扇区；对 GDPVal 训练–测试文件级重叠为 0。Qwen3.6-27B + SFT：GDPVal OpenHands **1445.7（+65.7）** / Codex +63.4；Claude Code 下 Workspace-Bench-Lite **63.7（+7.7）**、SpreadsheetBench II **24.0（+13.7）**；35B-A3B 亦大幅上涨。轨迹全在 Codex 上采、评测跨脚手架仍涨——教的是可迁移工作技能而非脚手架习惯。RFT：同候选池上，**锚定 judge 选轨**在 Workspace/Spreadsheet 上优于无锚/随机；GDPVal Elo 噪声大、未统计分离。HF：https://huggingface.co/collections/groundhogLLM/graphforge。

## 论文信息

- **标题**：GraphForge: Training Working Agents with Graph-Anchored Workspace Synthesis
- **作者 / 机构**：Qisheng Su 等（USTC / Shanghai Innovation Institute）、Hanchen Wang 等（Fudan）、Tao Gui（上海 AI Lab）等
- **数据 / 模型**：https://huggingface.co/collections/groundhogLLM/graphforge
- **链接**：https://arxiv.org/abs/2609.38923 · https://arxiv.org/pdf/2609.38923
- **观察时间**：2026-10-02（HF Daily Papers；working agent · evidence-anchored rubric · data synthesis）

---

## 🎯 为什么这件事值得写

Working agent 评测栈（GDPVal / Workspace-Bench / Claw-Eval / SpreadsheetBench II）已经逼近「真职业交付」，训练数据却还在「假文件」与「无 verifier」两极摇摆。GraphForge 的关键设计不是更大教师，而是 **任务与验证同源**：都从同一张证据图编译，criterion 必须声明读哪些文件、核对什么量。这直接服务 Byron 的 judge 纪律——**Rubric 先于 LLM 偏好、证据先于印象分**；也对照 [LibraryDesignBench](./20261001_LibraryDesignBench_论文解读_库写对了不算_下游agent写出来才算_为人API不等于为agent.md)：那边测「为人 API ≠ 为 agent」，这边造「为 agent 可核验的工作区任务」。

## 🏗️ 机制：种子控方向，图编译规格，一步修订再准入

**种子。** O*NET 职业 + 官方 work activity 先固定方向与执行模式，避免模型自由出题坍缩到高频occupation。

**Workspace。** Agent 按覆盖目标爬真实文件；文件带隐藏角色（core / supporting），平均每任务约 23 个输入文件，PDF 出现在 96.7% 轨迹中。

**Evidence graph → 任务 + Rubric。** 跨文件关系图编译成任务陈述与私有评分准则；每条正/负准则绑 evidence anchors（例：FY2025 revenue 锚 F2，并与 F1/F5 交叉验证）。

**One-step revision。** 教师先跑一轨；revision 对照原文件检查自然性、可执行性、数量是否有据、锚是否正确、核验说明是否够。任务陈述未变则可复用初轨，只改 rubric 绑定时不重跑——漏斗里 81.6% 不变 rollout 被复用。

**Admission。** 确定性检查（文件名/格式/表格公式等）+ 证据锚定 agent judge：
\[
Q_i(\tau)=\frac{\sum_{k\in R_i^+} w_{ik}^+ a_{ik}(\tau)-\sum_{j\in R_i^-}\lambda_{ij}v_{ij}(\tau)}{\sum_{k\in R_i^+} w_{ik}^+}
\]
再滤掉病态工具行为；\(Q>0.90\) 才进 SFT（3,638→2,169）。

## 🧪 关键证据

**主表（Table 2）。** 27B SFT：GDPVal 1380.0→1445.7（OpenHands）、1364.0→1427.4（Codex）；Workspace-Lite 56.0→63.7（Claude Code）；Spreadsheet II 10.3→24.0。35B-A3B：GDPVal +101.7 / +101.4 Elo，Spreadsheet +16.5。仍远低于 Claude Opus 5 / Qwen3.8-Max 前沿，但对开源 working agent 是实质跳跃；相对 Nex-N2-Mini-35B 亦全面领先。

**跨脚手架。** 训练轨迹仅 Codex；评测 OpenHands / Codex / Claude Code 全涨——反「背脚手架 API」。

**RFT（Table 3）。** 锚定选轨：Workspace/Spreadsheet 增益最大；随机选轨 GDPVal 掉约 9 Elo；无锚在 GDPVal 点估计更高但 CI 未分清——作者诚实写成「GDPVal 220 题上 Elo 差太吵，锚定优势主要显在 Workspace/Spreadsheet」。

**污染审计（Table 4）。** 39,201 训练文件 vs 260 GDPVal 文件：**0 共享**；occupation 仅覆盖 13/44；未覆盖职业上 SFT 胜率不更低（0.739 vs 0.692）——增益不像背题。

**Judge 敏感性。** 删掉准则引用的工作表 → 该准则 \(Q\) 平均 −0.377，非目标准则几乎不动；但行/数字/引用细粒度腐蚀仅 ~0.02 变化——**结构证据靠谱，精细数值核验仍受 judge 模型能力限制**（文中归因 GLM-5.2）。

**泼冷水。** 2.1K 规模相对 SWE-smith 等仍小；judge 细粒度弱；职业分布受准入过滤扭曲；GDPVal RFT 结论噪声大。

## 🔬 最有意思的部分

1. **任务与 Rubric 同源编译。** 消灭「题是一套、分是另一套印象」的经典造假空间。
2. **种子 vs 文件分工。** 多样性靠 taxonomy，真实性靠 crawl——比「模型同时编故事和文件」稳。
3. **Revision 只修规格、尽量不重跑。** 合成成本控制的工程细节，值得抄进数据管线。
4. **锚定 judge 的 RFT 信号。** 同候选、同预算，选轨策略决定继续涨还是随机伤——对齐「golden+rubric 闭环」。
5. **精细腐蚀测出 judge 上限。** 提醒：证据锚定是必要非充分，数值级核验仍要程序检查补刀。

## 🤔 我的判断

定位：**面向 working agent 的证据图锚定数据合成框架 + 开源轨迹/模型**（测量与数据基础设施，附带可训练收益）。亮点三：

1. 真文件 + 图编译 rubric，同时解决真实性与可验证性；
2. 跨脚手架转移与污染审计做得比多数合成论文诚实；
3. 锚定 vs 无锚 vs 随机的 RFT 对照，直接服务 judge/evals 工程。

局限：规模、judge 细粒度、职业偏斜、前沿差距仍大。对 Byron：**(1)** 造 working-agent 数据时，**每条 rubric 必须带 evidence anchors**，否则 RFT/录取只是换皮偏好；**(2)** 对齐 judge 闭环：程序检查文件存在/公式 → 锚定 LLM 判内容 → 细数值另挂脚本（呼应「代码检查先于 LLM 裁判」）；**(3)** OpenClaw / Hermes 类持久助手训练，优先抄「种子控分布 + 真 workspace」而不是纯合成沙盒；**(4)** 与 [Skill-Use](./20261002_Skill-Use_论文解读_技能挂上了不等于会用_Trigger合规边界才测真使用.md) / [MidHarness](./20261002_MidHarness_论文解读_别再只扩整条轨迹_动作边界采样验证才救终端agent.md) 连读：那边测执行纪律，这边供可核验考场。

一句话收束：**假文件骗不过职业交付，无锚 rubric 验不准交付物——GraphForge 证明：用真文件证据图同时编译任务与评分，working agent 才有可训练、可迁移的数据底座。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
