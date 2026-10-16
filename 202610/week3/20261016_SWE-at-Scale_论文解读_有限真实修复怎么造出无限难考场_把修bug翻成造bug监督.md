---
month: 202610
week: 3
date: 2026-10-16
type: 论文解读
slug: SWE-at-Scale
---

# 有限真实修复怎么造出无限难考场：把「修 bug」翻成「造 bug」监督

你有没有过这种体验：coding agent 评测与训练都喊「要更多可验证 SWE 题」，可真实 GitHub issue 按开发节奏慢慢冒，手搓环境又贵；一改用 LLM prompt 往代码里塞缺陷，造出来的要么是 `pass`/`TODO`/`return None` 那种假伤，要么反复撞上模型预训练里最熟的那几类 bug——刷多了 Pass@1 看起来不难，却不像真人会写坏的仓库。**真正卡脖子的，难道不是「通用模型会不会造 bug」，而是「有没有一份能反复采样的、来自真实修复机制的 bug 诱导分布」吗？**

上海交大、华北电力、清华、华东师大 / 上海创智、新加坡管理大学的 Silin Chen、Zihan Zhang、Xiaodong Gu 等这篇 **SWE-at-Scale**（arXiv:2610.21359）把答案写成一条反向管道：把历史 bug-fix commit **倒过来**变成 (golden → buggy) 监督，用 Qwen3.5-9B LoRA 训出可复用的 bug 诱导模型，再在新 golden 仓上采样、P2F 过滤、写成 GitHub 风格 issue。看完我的感受：这是把「合成 SWE 考场」从 **prompt 启发式**推到 **可训练的、可随采样预算扩多样性的机制分布**——Pass@1 反而更低，才是「题更硬」的信号。

## 核心摘要

SWE-at-Scale 从 SWE-Fixer / SWE-rebench / SWE-bench-Live / SWE-Gym 等来源收集修复实例，排除与 SWE-bench Verified 同仓重叠后，经质量过滤、双向 diff 往返校验、分布感知采样与人工抽检，得到 **13,407** 条真实修复变换，覆盖 **688** 个仓库。两变体同骨干：SWE-at-Scale 直接学统一 diff 的 bug 诱导补丁；SWE-at-Scale+ 先用 DeepSeek-V4-Flash 教师写出执行器就绪的自然语言 bug pattern（类别、位置、当前行为、改法、约束、可观察效应），再训 9B 预测 pattern，由执行器按「普通改代码指令」落地（刻意不揭示「这是在造 bug」）。任务装配：生成补丁 → `git apply` → 至少一条原先通过的测试变红（P2F）→ LLM 用 diff + P2F 源码与输出写 issue。对人偏好：相对真实 Verified，HPR **55.9% / 61.2%**；Usability **72.6% / 94.0%**。四解算器 Pass@1：SWE-at-Scale **31.68–48.48%**，SWE-at-Scale+ **16.81–37.02%**，整体低于 SWE-smith / BugPilot 合成基线，+ 变体逼近专家设计的 DeepSWE 难度。同文件反复采样到 K=200：MBTE 从 0.583 升到 **2.063** bit，可用补丁中 **88.27%** 经归一化 diff 去重后仍唯一。价值定位：**可验证 coding-agent 评测 / 训练数据的可扩展合成基础设施**——把有限真实修复变成可复用的 bug 诱导监督。

## 论文信息

- **标题**：SWE-at-Scale: Scalable Issue Synthesis from Historical Bug Fixes
- **作者**：Silin Chen, Zihan Zhang, Xiaodong Gu（通讯）, Jiale Huang, Yuling Shi, Han Li, Zhao Shan, Chengcheng Wan, Zhensu Sun, Haibing Guan
- **机构**：Shanghai Jiao Tong University · North China Electric Power University · Tsinghua University · East China Normal University / Shanghai Innovation Institute · Singapore Management University
- **代码**：https://github.com/cslsolow/SWE-at-scale
- **模型 / 数据**：https://huggingface.co/SWE-at-Scale
- **链接**：https://arxiv.org/abs/2610.21359 （v1 2026-10-15；HF Daily Papers 2026-10-16）

---

## 🎯 为什么这件事值得写

SWE 考场要同时满足：真实仓上下文、可观察行为缺陷、像样的 issue 描述、可在原测试基建上执行。天然 issue 跟不上 agent 迭代；合成路线（SWE-smith 的 LLM-Modify / Rewrite、BugPilot FeatAdd、SWE-Synth 等）证明 LLM 能造出可执行 bug，但多样性仍绑在「通用模型里已有的造 bug 先验 + 提示词」上——同一上下文多采几次，容易塌回熟悉模式。

这和 [WorkForge](./20261015_WorkForge_论文解读_可验证工作环境也能规模化_真实工作区事实锚定任务与裁判.md)「可验证环境也要能规模化」、[SWE-Duel](../week2/20261013_SWE-Duel_论文解读_静态榜会漏进训练又被刷穿_让agent互出题互修难度跟着作者一起涨.md)「静态榜会被刷、难度要跟着作者涨」、[ProgramGym](../week2/20261013_ProgramGym_论文解读_从零复现程序缺可验证考场_1648黑盒环境加角色隔离把27B抬过旗舰脚手架.md) / [SoftwareGym](../week2/20261012_SoftwareGym_论文解读_一个OSWorld应用都没训_合成182个桌面软件让9B模型涨15点.md)「合成可执行考场」是同一条供给链问题：**评测与训练都缺「够真、够硬、还能继续长」的题。** SWE-at-Scale 的赌注是——**历史修复本身就编码了真实失败机制；倒过来训，比反复问通识模型「请注入一个 bug」更像开发现场。**

| 路线 | 监督从哪来 | 多样性瓶颈 |
| --- | --- | --- |
| 真实 SWE-bench 族 | 自然 issue 节奏 | 仓与人工策展成本 |
| Prompt 注入 / 重写 / 加功能回归 | 通用 LLM 先验 | 重复采样易塌模式 |
| **SWE-at-Scale** | 真实修复的反向变换 | 训练仓覆盖 + 推理采样预算 |

## 🏗️ 机制：反向修复 → 训诱导模型 → P2F 装配

**Stage 1：数据。** 对每条真实修复：先把 golden patch 打回 buggy 仓，再算「从已修文件回到原 buggy」的统一 diff，作为 bug 诱导目标。过滤缺补丁、非代码、以 import/注释/空白为主、结构不完整的样本；双向 apply 往返校验；按 BugPilot 风格标 bug 类型；超 64K 序列丢弃；分布感知平衡类型 / 仓 / 上下文长度后再人工抽检。

**Stage 2a：SWE-at-Scale（直接补丁）。** 输入带行号的完整 golden 文件，输出 unified diff；模型同时学「改哪里」和「怎么改坏」。推理时对任意正确仓文件采样，温度控制探索广度，`git apply` 后进执行校验。

**Stage 2b：SWE-at-Scale+（pattern → 执行器）。** 教师模型把每条已验证反向变换写成可执行 NL pattern；9B 学预测 pattern；执行器只当「按说明改代码」，降低执行器自身「我在造 bug」先验的干扰。作者认为显式化 bug 逻辑更贴 LLM 语言建模能力——对应更高 Usability / HPR。

**装配与验证。** 保留至少一条 P2F；再用 LLM（diff + P2F 测试源与输出 + 两条 demonstration）写用户向 issue，不暴露合成过程。对照实验里，各方法在同一 Verified 关联目标文件 / golden 状态上造题；解算用 mini-SWE-agent（温度 0、最多 250 动作），且 **model-disjoint**：写 issue 的模型不参与解同一批题。

训练细节：Qwen3.5-9B，LoRA rank 16 / α 32 / dropout 0.05，1 epoch，64K 上下文，单卡 RTX PRO 6000，有效 batch 8，lr \(1\times10^{-4}\)。

## 🧪 关键证据

**真实感 × 可用性（Figure 5）。** SWE-at-Scale+：**94.0%** Usability、**61.2%** HPR；SWE-at-Scale：**72.6% / 55.9%**。最佳 LLM-Rewrite 约 **72.2% / 51.8%**；LLM-Modify 人偏好明显更低；FeatAdd 真实感尚可但可用性因「加功能是否碰巧回归」弱控而差。HPR 协议：三名 ≥3 年经验工程师同上下文对比合成 vs 真实 Verified，多数票。

**难度（Table 1，Pass@1 %）。** SWE-bench Verified 上四解算器约 69.6–97.0；DeepSWE 约 24.3–63.4。合成基线普遍更高分（更容易）。SWE-at-Scale：DS-V4-Flash **48.48**、GPT-5.4-mini **31.68**、GLM-5.3-Flash **42.15**、Qwen3.8-Flash **44.44**。SWE-at-Scale+：**33.19 / 16.81 / 35.32 / 37.02**——全场最低，逼近 DeepSWE 量级。作者强调：更低 Pass@1 是跨解算器一致趋势，不是单一 solver 偏见。

**多样性缩放。** 固定 2,000 条、只变训练仓数（5→100）：MBTE 随仓覆盖升，可用性稳中有升。温度：MBTE 在 0.4–0.6 峰值（约 **1.153–1.173**），温度 1.0 时可用性掉到 **23.4%**、熵回落。采样预算 \(K\in\{5,25,50,100,200\}\)、t=0.6、50 文件：MBTE **0.583→2.063**；K≥50 可用性约稳在 **47%**；K=200 时 4,749 可用中 **4,192（88.27%）** 唯一。对照 LLM-Modify+DS-V4-Flash：同设 MBTE 仅 **1.22**、唯一率 **61.74%**。

**复杂度与成本。** 平均改动行数：SWE-at-Scale / + 约 **18.8 / 18.5** vs Verified **14.3**；直接路径约 **17.45k** token/实例，低于 Rewrite（约 35–63k），远低于 FeatAdd（约 **2.29M**）。

**泼冷水。** 目前只做 Python；弱测试覆盖仓上，测不到的真实缺陷会被 P2F 丢掉；Usability 定义含「五次内至少一条非平凡 P2F」，与「无限采样下的唯一性」是不同轴；HPR 是相对同上下文真实题的偏好，不是绝对「像生产」分数。

## 🔬 最有意思的部分

1. **「Pass@1 更低」被当成成功指标。** 合成题若只报可解率，会奖励「假容易」；本文把难度与 Usability / HPR 并列表，避免「造得多但像玩具」。
2. **+ 变体：把 bug 逻辑外化成 NL。** 直接 diff 把「造什么坏」压在补丁里；pattern 路径给出可审计的中间表示，也解释了为何 Usability 冲到 94%。
3. **同文件多类型，而不是靠堆文件刷类型直方图。** MBTE 专门卡死「换文件假装多样」；K=200 仍涨熵，说明学习到的是可探索分布，不是有限模板表。
4. **训练仓覆盖 vs 样本量解耦。** 同 2k 点、更多仓 → 更高 MBTE：对 harness 数据策展是直接提醒——**异构修复机制比同仓堆量更值钱。**
5. **和 SWE-Duel / WorkForge 互补。** Duel 让 agent 互出题抬难度；WorkForge 钉工作区事实；本文钉的是 **修复机制的反向分布**——三者都在回答「题从哪来、如何不塌」。

## 🤔 我的判断

定位：**可验证 SWE 任务合成的方法 + 数据基础设施**，对准 coding-agent 评测供给与可执行训练数据，而不是再刷一条解算器榜。

亮点三条：

1. 把真实修复倒成可训练的 bug 诱导监督，跳出「通识模型 + 提示词」天花板；
2. 直接补丁与 NL pattern 两路径对照，显式化逻辑换来可用性与人偏好；
3. 用 MBTE + 采样预算曲线证明多样性可在推理时继续扩，且不快速塌缩。

问题与局限：Python-only；P2F 绑定测试质量；合成仍依赖 LLM 写 issue，issue 文风与人类报告的差距未系统测；解算器集合偏 Flash / mini 档，长程 agent harness 差异未展开。

对工程的直接启发：**(1)** 造 SWE 训练/评测数据时，优先挖 **已验证修复的反向变换**，而不是无限 prompt 注入；**(2)** 报表同时给 Usability、人偏好、跨解算器 Pass@1，别只晒「造了多少条」；**(3)** 扩多样性先扩 **训练仓覆盖**，再调温度与 K；**(4)** 考虑把 bug pattern 当中间产物——可审计、可过滤、可交给不同执行器；**(5)** 与 [AgentSWE](../week2/20261014_AgentSWE_论文解读_coding_agent交出来看着齐其实空_冻结提交加holdout_rubric才测得出真交付.md) 式 holdout 交付验收叠用时，合成题更适合当 **训练/筛选池**，最终对外宣称仍要仓外 holdout。

一句话收束：**有限的真实修复，一旦被训成可采样的 bug 诱导分布，就能在新仓上下文里持续长出更真、更硬、且不那么容易塌缩的可验证 SWE 考场。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
