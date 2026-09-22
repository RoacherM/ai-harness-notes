---
month: 202609
week: 4
date: 2026-09-22
type: 论文解读
slug: CodeMidas
---

# 只靠源码，榨出五千可验证 coding RL 环境：CodeMidas 的 Midas 之触

你有没有过这种体验：想训 coding agent，却卡在「有仓库、没任务」——现成管线大多要 issue / PR / commit / 已有测试或文档，覆盖到哪，任务就只能长到哪。小米 LLM Core 等这篇（arXiv:2609.22068；亦见今日 HF Daily Papers）提出 **CodeMidas**：用**源码本身**当唯一任务专用输入，把已实现功能自动变成可执行 RL 环境。看完我的感受是：这是 agentic coding 的数据飞轮件——不跟开发痕迹绑死，从「代码里已经有什么」反向生成「让 agent 再实现一遍」的可验证任务。

## 核心摘要

CodeMidas 是一条 agentic 管线：探索已实现功能 → 写成行为规格 → 以**执行原码** grounding 构造测试 → 经执行一致性与 solution rollout 校验/过滤。得到 **5,545** 条训练任务，来自 **3,185** 个开源仓库，覆盖 **23** 语言、**15** 技术领域；前十大语言覆盖 **5,445/5,545（98.2%）** 任务。用 GRPO 训 MiMo-V2.5，五个外部基准全涨：DeepSWE **10.0%→21.7%（+11.7）**，ProgramBench Almost Solved **4.5→21.5（+17）**，Terminal-Bench v2.1 **+8.5**（63.7→72.2）；另有 SWE-bench Pro、RepoZero C2Rust 增益。规模消融：高质量任务 **1k→3k→5,545** 逐步更好；高质量 3k 已全面胜过未清洗的 vanilla **8k**。轨迹上：SWE-bench Pro 交互轮次 **37.3→50.1**，ProgramBench **155.1→122.8**——探索变多、整机构造反而更短更有效。

## 论文信息

- **标题**：CodeMidas: Scaling Agentic Coding RL Environments from Code Itself
- **作者**：Bowen Ye*、Lei Li、Shicheng Li、Zihao Yue、Linghao Zhang、Hanglong Lv、Yuanxin Liu、Wenhan Ma、Hao Tian、Rang Li、Jinhao Dong、Yikai Zhao、Xiangwei Deng、Hailin Zhang、Liang Zhao、Qi Liu、Lingpeng Kong、Tong Yang†、Fuli Luo†（*实习于小米；†共同通讯）
- **机构**：Xiaomi LLM Core · 北京大学 · 香港大学 · 中国人民大学
- **链接**：https://arxiv.org/abs/2609.22068 （v1，2026-09-18）
- **观察时间**：2026-09-22（HF Daily Papers）

---

## 🎯 为何「只靠源码」是缩放关键一跳

SWE-bench 系从 issue 出发；R2E-Gym 等靠 commit；SWE-smith / SWE-Flow / SWE-Hub 缠着已有测试；R2E / MindForge 靠文档或编译参考。表 1 对比很刺眼：多数管线至少有一类 ✗（必须 issue/PR/commit/tests/description）。CodeMidas 把五列全标成「不要求」，语言覆盖拉到 **23**——主张是：已实现功能同时提供任务种子与候选解；公开接口与可观测行为定义「该实现什么」；执行原码给测试期望；外围仓库改造成「功能待补」的开发起点。缩放瓶颈从「有多少带痕迹的变更」换成「有多少可执行功能可以被说清楚并验出来」。

## 🏗️ 管线：设计 → 测 → 一致 → 滚动过滤

漏斗（文中示意数量级）：候选从约 **22,575** 压到最终 **5,545**，中间经测试构造、执行一致性、泄漏检查、方案审计、rollout 结局过滤。

1. **Task design**：agent 探索功能，写出行为规格；从仓库摘掉/改写目标实现，留下真实依赖与结构。
2. **Test construction**：从任务派生输入，对**参考实现**执行得期望输出，再断言候选解同输入同输出——测试接地在原码执行，而非纯模型臆造。
3. **Execution consistency**：空实现应稳定失败、参考解应稳定通过；两边任一侧不稳就丢。
4. **Post-rollout filtering**：对抗 rollout 探可利用泄漏；审查 agent 核对「陈述 vs 验证器」的 FP/FN；只保留在固定预算下**既有成功也有失败**尝试的任务（全过/全挂信息量差，且可能藏缺陷）。

数据集侧：Python / TypeScript / Go 居前（约 21.4% / 18.3% / 16.2%）；Systems / Web / Dev Tools 三大域合计约 45.6%；参考补丁中位 **142** 行——不是玩具函数题。

## 🧪 RL 结果：五榜全涨，质量胜过堆量

在 5,545 任务上 GRPO 训 MiMo-V2.5：

| 基准 | 变化（文中） |
| --- | --- |
| DeepSWE | 10.0% → 21.7%（+11.7） |
| ProgramBench（Almost Solved） | 4.5 → 21.5（+17） |
| Terminal-Bench v2.1 | 63.7% → 72.2%（+8.5） |
| SWE-bench Pro / RepoZero C2Rust | 文中分别标 +4.1 / +11.3 量级增益 |

消融（SWE-bench Pro、DeepSWE、CodeMidas Val）：高质量 1k / 3k / 5k 曲线递升；**高质量 3k 全面压过 vanilla 8k**；满量 5k 相对 vanilla 8k 在三评上分别高出约 0.59 / 4.59 / 4.49 pp。结论很 harness：**过滤后的可验证任务密度，比粗糙堆 8k 更值钱**。

## 🔬 行为变化：探索与自验证，不只是分数

训练后 agent 更愿逛仓库、自写检查更多样；写检查与更高成功率相关。外推到外部任务：SWE-bench Pro 轮次 **37.3→50.1**（issue 修复探索变长）；ProgramBench **155.1→122.8**（整机构造探索升、总交互反而缩短）——说明「更好的行为」不是无脑加长，而是按任务形态重配预算。奖励侧：CodeMidas 用合成测试的执行奖励 + GRPO，不上 reward model——把「可验证」压到环境层，而不是再训一个爱漂移的裁判。


## 📈 和「从仓库榨技能」叙事的并排

同月归档里已有从海量仓库蒸馏技能库的路线（如 Code2Skill）；CodeMidas 拧的是另一颗螺丝——不是萃取文档化 skill，而是把**可执行功能**变成带 verifier 的 RL 课表。工程上两者可并联：skill 库降低探索成本，CodeMidas 式环境提供可强化的成功信号。关键纪律不变：奖励尽量执行接地；过程 rubric 若必须上 LaaJ，先过 RuVerBench 式校准；确定性脚手架层用 Layer-Isolated 式切片闸门守住回归。

## 🤔 我的判断

定位：**从源码本身规模化 agentic coding RL 环境的数据飞轮**，补上「无 issue 也能造任务」这一环。亮点三：

1. 任务专用输入收敛到 source code，覆盖语言/域显著宽于痕迹驱动管线；
2. 执行 grounding + 多层过滤，把「能跑的测试」当成一等公民，而不是生成完就训；
3. 质量>数量的消融与行为分析，证明飞轮转的是**可验证难度曲线**，不是 raw count。

局限：行为规格与审查仍含模型环节，可能引入分布偏置；全过/全挂过滤会丢掉极难/极易尾；执行奖励依赖测试充分性（EvalPlus / PatchDiff 类风险仍在）。对 Byron：若你在攒 coding-agent 训练/评测资产，可把 CodeMidas 式管线接在 harness 外环——**(a)** 仓库 → 功能探索 → 规格 → 执行接地测试 → rollout 过滤；**(b)** 训练前先过一致性与泄漏门；**(c)** 规模优先「高质量子集曲线」而非 raw 8k；**(d)** 与同日 [RuVerBench](./20260922_RuVerBench_论文解读_裁判验长轨迹Rubric仍有噪声.md) 对照：能执行验证的别交给 LaaJ，LaaJ 留给执行验不了的过程 rubric，且要 meta-eval。

一句话收束：开源代码里已经藏着海量「可再实现一遍」的功课——CodeMidas 做的是把它们炼成**带可靠验签的 RL 环境**，让 coding agent 的数据飞轮转起来。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
