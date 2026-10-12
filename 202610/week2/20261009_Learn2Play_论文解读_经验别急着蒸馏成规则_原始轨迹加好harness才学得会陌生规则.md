---
month: 202610
week: 2
date: 2026-10-09
type: 论文解读
slug: Learn2Play
---

# 经验别急着蒸馏成规则：原始轨迹 + 好 harness，agent 才学得会陌生规则

你有没有过这种体验：给 agent 挂上 Reflexion、ReasoningBank、AWM 一类「自进化」记忆，跑了几轮，曲线确实在涨，可你心里一直犯嘀咕：它是真的**从交互里学到了新规则**，还是只是把预训练里早就知道的东西又想起来了？SWE、Web、桌面任务都太「眼熟」，涨分根本分不清是学习还是回忆。NUS Bryan Hooi 组这篇 **Learn2Play Bench**（arXiv:2610.08215）专门拆这个问题：**在规则陌生甚至反直觉的环境里，LLM agent 到底能不能从经验中学？**

看完我的感受：这是一张给「持续学习 / self-evolving」降温的体检表。最朴素的「把完整原始轨迹留在上下文」赢了一堆花哨的反思与规则抽取；同一个模型换个 harness，分数和学习斜率一起涨，钱还更省。

## 核心摘要

Learn2Play Bench 手工设计了 **20 个全新的文本游戏模板**，隐藏规则要么新、要么故意反常识（例如「偷同事午饭才能升职，努力工作没用」），agent 只能靠行动与文本反馈去发现规则，**不更新权重**、跨 episode 携带经验。游戏反馈确定、自动打分、可回放；其中 10 个固定实例、10 个「re-shuffled」实例（角色、物品、布局变，核心规则不变），专门测迁移；另有 5 个挑战游戏用 10 局窗口、权重 3。指标拆两类：表现（Max、Mean）与学习（Learning Gain、Learning Slope）。实验覆盖 OpenCode 下 11 个 backbone、6 种自进化方法（Naive / Memory / Reflexion / EvoTest / ReasoningBank / AWM）、Claude Code 与 Codex 两个部署 harness，以及每局 21 到 30 名大学生的人类对照。主结论三条：**保留原始证据比压缩成规则更利于学习**（Kimi K3 上 Memory 四项指标全胜）；**人类 Top-1 峰值仍高于所有 agent**（84.3 对 Opus 5.5 的 80.1），人类探索更多、重复更少；**同 backbone 换 harness 能同时提分降本**（Opus 5 从 OpenCode 换到 Claude Code，Max 74.5 到 81.8，LG +21.3 到 +27.1）。它的价值不在提出新方法，而是第一次把「学习」与「回忆」在 agent 评测里干净地拆开。

## 论文信息

- **标题**：Learn2Play Bench: How Well Do LLM Agents Learn from Experience in Unfamiliar Environments?
- **作者**：Yibo Li, Jinhang Qiu, Zhi Zheng, Qianyun Guo, Jiaying Wu, Shuo Ji, Bryan Hooi
- **机构**：National University of Singapore
- **项目页**：https://liushiliushi.github.io/learn2play-bench-website/
- **链接**：https://arxiv.org/abs/2610.08215 （v2，2026-10-08；HF Daily Papers 2026-10-09）

---

## 🎯 为什么这件事值得写

自进化 agent 这两年方法层出不穷：Reflexion 写反思，ExpeL 抽 insight，Voyager 存可执行技能，AWM 归纳 workflow，ACE 维护 playbook，EvoTest 连 prompt 和状态提取程序一起进化。配套基准也不少：StreamBench、Evo-Memory、LifelongAgentBench、SWE-Bench-CL、MemoryArena……但它们有一个共同的盲点：**任务大多来自现实世界熟悉的场景**。模型在第三轮做得更好，可能是学到了，也可能只是「想起来了」。J-TTL 用 Jericho 游戏，但那些游戏的攻略本身就在预训练语料里；ARC-AGI-3 够陌生，可又掺进了视觉能力。

Learn2Play 的切口很窄也很准：**纯文本、规则全新、反馈确定**。这样涨分只能来自交互中获得的新知识。对做 skills / memory / harness 的工程师来说，这正好是一个干净的「经验管线 A/B 台」。

## 🏗️ 机制：四条设计原则与四个指标

设计原则：

| 原则 | 做法 |
| --- | --- |
| 规则新颖 | 隐藏规则不来自已有游戏，部分刻意反常识 |
| 自动打分 | 每局终局自动出分，无需人工评 |
| 可复现 | 确定性结果，可回放、可逐步对照 |
| 受控变化 | re-shuffled 游戏换表象不换规则，测迁移 |

协议：agent 走黑盒命令接口，看不到源码与 oracle；普通游戏 5 局、挑战游戏 10 局；每个游戏 3 次独立 trial，trial 之间清空记忆，trial 内保留什么由 harness 决定。

指标（分数先归一化）：

$$
\mathrm{LG}_g = \frac{1}{k_g}\Big(\sum_{t=T_g-k_g+1}^{T_g} y_{g,t} - \sum_{t=1}^{k_g} y_{g,t}\Big),\quad k_g=\min(3,\lfloor T_g/2\rfloor)
$$

LS 是对每局均分做线性回归的斜率。Max / Mean 衡量「做得多好」，LG / LS 衡量「学得多快」。跨游戏聚合时挑战游戏权重 3，使 5 个挑战游戏与 15 个普通游戏各占一半。

## 🧪 关键证据

**backbone（OpenCode 统一 harness）。** Opus 5.5 拿下 Max 80.1 / Mean 61.0；但学习最快的是 Qwen 3.8 Max（LG +25.2，LS +6.30）。**分最高的不等于学得最快**。GPT-5 Mini 垫底（Max 34.0，LS +1.83），日志里能看到它在 Poisoner 里按菜单位置点菜，完全不用反馈。

**自进化方法（固定 backbone）。** Kimi K3 上：Memory（原始轨迹全塞）Max 63.3、LG +16.9，四项全胜；EvoTest 虽然 Max 62.4，LG 只有 +1.8；Naive 垫底。Opus 5 上 EvoTest 与 Reflexion 领先，Memory 居中，AWM 最差（它只用成功轨迹，失败不入库）。更关键的是：**OpenCode 本身就比这些方法强**，因为它让模型自己决定何时回看反馈、跑计算、核对计划。

**harness（同 backbone）。**

| 配置 | Max | Mean | LG | LS |
| --- | --- | --- | --- | --- |
| Opus 5 @ OpenCode | 74.5 | 53.8 | +21.3 | +5.13 |
| Opus 5 @ Claude Code | 81.8 | 62.3 | +27.1 | +6.59 |
| GPT-5.6-SOL @ OpenCode | 57.0 | 41.1 | +14.5 | +3.71 |
| GPT-5.6-SOL @ Codex | 74.3 | 54.8 | +22.2 | +5.66 |

GPT-5.6-SOL 换到 Codex，Max 涨 17 分以上。成本上，OpenCode + Kimi K3 比 Memory 少用 56% 输入 token，比 EvoTest / ReasoningBank 少生成 71% / 41% token，原因是一次回复可执行多个动作、并去掉重复反馈。

**迁移（re-shuffled）。** 表象一变，Sonnet 5 的 LS 从 4.90 掉到 2.23，Gemini 3.7 Flash 从 4.49 掉到 2.34；Opus 5 只掉 0.59，Claude Code 版 Opus 5 掉 0.52。弱模型「记住了什么管用，却没记住为什么管用」。

**泼冷水。** 20 个游戏都是文本选项式，动作空间受限；人类样本是大学生，每局 21 到 30 人；backbone 与方法交叉不全（方法对比只在三个模型上）；harness 优势部分来自「模型就是在自家 harness 上训的」，作者也承认这一点。

## 🔬 最有意思的部分

1. **规则蒸馏会把不完整证据固化成禁令。** GemForge 里 EvoTest 一次组合失败后，进化出的 prompt 直接禁止了一批没试过的组合，其中就有能成功的那一个。原始轨迹的好处是：结论错了还能回头翻证据改。
2. **会背规则不等于会规划。** Haunted Inn 里 Opus 5 能复述所有禁忌，却贪心分房，最后一位客人无房可住；下一局才学会给选项少的客人预留。
3. **人类赢在探索。** 固定游戏里相邻局动作序列相似度人类 0.54、agent 0.66；低分局之后下一局刷新个人最佳，人类 33%、agent 22%。agent 找到「能用的」就不再找「更好的」。
4. **agent 赢在执行与计算。** RedEye、Model Auditor 这类需要长序列执行或写程序验证假设的游戏，agent 反超人类；部分人类后期懒得再测。
5. **贵 token 不等于贵 episode。** Kimi K3 单价高于 Sonnet 5，但每局 0.56M token 对 0.73M，总成本更低、分更高。

## 🤔 我的判断

定位：**持续学习 / self-evolving 的测量基础设施**，核心贡献是把「从交互中学」和「调用预训练知识」分开。

亮点三条：

1. 反直觉规则 + 确定性反馈 + 可回放，第一次让「学习曲线」有了干净的因果解释；
2. 同时扫 backbone、经验方法、harness 三个维度，并给出成本前沿，结论能直接落到工程选型；
3. re-shuffled 对照把「记住动作」与「记住原因」拆开，这正是技能库最常见的失效模式。

问题与局限：文本选项式环境离真实工具调用还有距离；方法对比覆盖的模型少；harness 优势与「模型和 harness 共训」混在一起，无法完全归因到 harness 设计。

对工程的直接启发：**(1)** 做 skills / memory 管线时，蒸馏出的规则要保留指向原始轨迹的引用，允许被新证据推翻，别让「一次失败」变成永久禁令，这和 [JITMem](../../202609/week4/20260925_JITMem_论文解读_记忆别在写入时裁剪_读时按任务现场策展同一条轨迹能榨出不同课.md) 读时策展的思路一致；**(2)** 失败轨迹必须入库，AWM 只收成功样本在这里吃了大亏；**(3)** 评估自进化方法前，先把 harness 调好再比，否则「方法增益」可能还不如换个 harness；**(4)** 技能文件里写「为什么有效」的条件，而不只写「做什么」，否则表象一换就失灵；**(5)** 给 agent 加显式探索预算，避免过早收敛到次优策略。

一句话收束：**Learn2Play 证明，在真正陌生的规则面前，最可靠的经验是没被压缩过的原始证据，最划算的升级往往是换一个更好的 harness。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
