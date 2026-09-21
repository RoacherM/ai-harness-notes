---
month: 202609
week: 3
date: 2026-09-21
type: 论文解读
slug: Subagents-vs-Skills
---

# 技能该塞进主上下文，还是开子代理？微软剑桥把同一包知识换了执行方式

你有没有过这种体验：SKILL.md 写得很完整，一 `Skill` 进主对话，长程任务越跑上下文越胀，中间推理开始糊——你怀疑是「技能内容不行」，其实可能是**调用方式**错了。Claude Code / Codex 都支持 subagent，但实务上多拿来并行降延迟，很少当成「可复用知识的封装执行」。Cornell / Microsoft Research Cambridge 这篇（arXiv:2609.09233）做了一个干净对照：**同一 skill package，要么当 agent-skill 灌进主上下文，要么当 subagent 开新窗口只回传输出。** 看完我的感受是：技能库争论常停在「写什么」，这篇把杠杆挪到「怎么组织、怎么调用」——和 Byron 关心的 skills / harness / MCP 接口设计直接同构。

## 核心摘要

作者指出：当前主流 agent skills（Anthropic 风格 skill package：描述 + `SKILL.md` + 资源）执行时只是把配方加载进主上下文，由主策略逐步跟着做；这和 RL 里「可调用的 temporally extended policy」不同。长程下上下文膨胀会拖垮推理质量。替代方案 **subagent execution**：不把指令暴露给主上下文，而是用 skill 指令初始化独立上下文，跑完只返回输出——换峰值上下文降低，付协调通信 token。关键条件：skill 需呈现为**带清晰 I/O 契约的程序性知识**（类比 options：initiation set / policy / termination）。在 SkillsBench（**87** 题；主结果落在为其中 **64** 题合成的契约化技能包）上：对**缺乏 I/O 契约**的人工策展包，agent-skill 各模型上匹配或优于 subagent；换成作者从成功轨迹合成的 **procedural + I/O contracts** 包后，趋势反转，**subagent 全面优于 agent-skill**，小模型增益更大。干扰工具增多时，subagent 退化更缓；强模型（如 GPT-5.3 Codex、Kimi-K2.6）上 **>80%** 任务峰值上下文更低（分模型任务占比约 **28.1%–95.3%**），总 token 则明显更高。结论：**可复用知识的收益取决于组织与调用方式，而不只是内容。**

## 论文信息

- **标题**：Subagents vs Agent Skills: Executing Reusable Knowledge for Long-Horizon Agentic Tasks
- **作者**：Wasu Top Piriyakulkij*（Cornell；MSR Cambridge 实习）、Rachel Lawrence、Alicia Curth、Sushrut Karmalkar、Niranjani Prasad（Microsoft Research Cambridge）
- **机构**：Cornell University · Microsoft Research Cambridge
- **链接**：https://arxiv.org/abs/2609.09233 （v1，2026-09-07）

---

## 🎯 为什么「有 skill」仍会在长程崩

Agent skills 已成领域知识注入的主流格式：名字与描述给发现，调用后加载 `SKILL.md`，主 agent 自己逐步执行。问题在于：它**并不执行技能**，只是把更多指令塞进同一窗口。Lost-in-the-middle / 上下文长度伤害等文献已说明：信息在、上下文胀，表现仍可掉。

现有 harness 的 subagent 多用在可并行子任务降延迟，子代理常常**不带 skill package**，只是轻量工人。本文把 subagent 重新定义为：**用独立上下文封装可复用程序性知识的执行机制**——信息封装视角下，主代理看不到子推理，只见最终输出。

代价也写清楚：多技能知识难以在同一子任务内混合；主–子之间要付通信开销；主代理必须会选对子代理并给出充足输入（额外的 delegation 问题）。

## 🏗️ 同一 package，两种算子

形式化很短但有用。Skill package \(s\) 含描述 \(d\)、指令 \(m\)（`SKILL.md`）、资源 \(R\)。

- **AgentSkill(\(s\))**：调用无额外参数，直接返回指令 \(m_k\)；后续每一步仍是主上下文里的普通 agent step——知识条件化的是**同一个** \(\pi_{\mathrm{LLM}}\)。
- **Subagent(\(s\))**：调用时带任务输入 \(x_t\)；新建上下文，以 \(m_k\) 为种子跑独立 tool-calling 过程，只把输出交还主代理——知识条件化的是**每个 package 一份**子策略 \(\pi^{(k)}\)。

适合 subagent 的包，描述写成：

\[
d = (h,\ q_{\mathrm{in}},\ q_{\mathrm{out}})
\]

\(h\) 用途摘要，\(q_{\mathrm{in}}/q_{\mathrm{out}}\) 自然语言 I/O 契约。对应 options：\(q_{\mathrm{in}}\)≈ initiation set，\(m\)≈ option policy，\(q_{\mathrm{out}}\)≈ termination / 返回约束。假设：**程序性指令 + 显式契约 → 可安全封装到子上下文。**

合成流程：用 OpenHands + GPT-5.3 Codex 对 SkillsBench 跑成功轨迹（三独立 run），再经 Copilot CLI（极少人工）合成契约化包，覆盖 **64/87** 任务；全文主结果在该子集。Harness 为 OpenHands（SDK 少量改动）。

## 🧪 主结果：契约一换，优劣对调

**Figure 2（核心）**：横轴多底座（Ministral-8B、Gemma-4-12B、Qwen3.5-9B、Mistral-Large-3.1、gpt-5.4-mini、Kimi-K2.6、gpt-5.3-codex 等），比较五种条件：无技能 / agent-skill 无契约（策展包）/ subagent 无契约 / agent-skill 有契约 / subagent 有契约。

- **无 I/O 契约（SkillsBench 人工包）**：内容偏「相关知识」，很少规定期望输入输出 → **agent-skill ≥ subagent**（全模型）。
- **procedural + I/O contracts（合成包）**：趋势反转 → **subagent > agent-skill**；带宽更紧的小模型增益最大。
- 合成包在两种执行模式下都优于策展包（非严格对照，内容不同，但说明合成质量至少可比）。

**上下文压力（Figure 3）**：往初始上下文追加无关 distracting skills/tools 描述。随干扰数升到上百（图中到 **263**），subagent 准确率掉得更慢——峰值上下文优势在「环境很吵」时更值钱。

**峰值 vs 总量（Figure 4）**：

- 左：subagent 峰值上下文更低的任务占比——弱模型约 **28.1% / 45.2%**，中等约 **59.4% / 32.8%**，强模型 **70.3% / 79.7% / 95.3%**；文中概括强模型上 **超过 80%** 任务峰值更低。弱模型「峰值不降」常因 agent-skill 早停、上下文没来得及涨，跨模式不可直接比。
- 右：subagent **总 token 显著更高**——信息要复制进各子窗口，通信开销是设计税。

## 🔬 最有意思的部分：库怎么长，比「再写一篇 SKILL.md」更重要

附录方向（A.1）开始碰 **skill-library organization**：扁平 vs 层次树/图；叶节点有清晰契约适合 subagent，上层「编排型」知识更适合 inline agent-skill。这和软件工程里「大仓如何分层」同构——结论段把未来工作点成 **skill-library refactoring**：可复用过程如何抽象、因子化、随库增长重组。

对工程的直接含义：

1. 不要默认「有 subagent 按钮 = 该把所有 skill 丢进去」；先问有没有 **I/O 契约** 与可独立执行的程序步骤。
2. 策展知识型 skill（概念、约束、多技能交织）继续 **agent-skill / 主上下文** 可能更稳。
3. 轨迹合成契约化包，是让 subagent「用得起来」的实用路径——和 [Code2Skill](./20260921_Code2Skill_论文解读_还没跑轨迹也能攒技能库_从两万仓库榨出百万条.md)「从代码挖技能」、[EvoSkill-GUI](./20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md)「部署时进化」互补：一个管**执行模态**，一个管**内容来源**。

局限诚实写在结论：subagent 只有在契约+程序知识到位时才赢；多技能融合能力变弱；通信协议仍开放（最小契约是一条路，非全部）。实验绑定 OpenHands + SkillsBench 子集，外推到 Claude Code / Codex 原生 subagent 产品行为需再测。

## 🤔 我的判断

定位：**技能执行模态的对照实验 + 接口设计论文**，不是新的技能学习算法。亮点三：

1. 把「同一 package、两种调用」做成可复现对照，结论可操作：无契约灌主上下文，有契约开子代理；
2. 用干扰工具把「峰值上下文」从口号变成可观测的 graceful degradation；
3. 明确总 token↑ / 峰值↓ 的交易，避免只吹封装不谈账单。

对 Byron 的栈：Wayne-Skills / MCP / Pi Manifest 里，skill 既是文档也是可分发单元——这篇建议在清单层区分 **inline skill** 与 **subagent skill**（描述里强制 `qin/qout`），并在评测里同时报准确率、峰值上下文、总 token，而不是只报 pass。和 [EvoHarnessBench](./20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) 连读更有味道：库扩张时 engagement 本就稀缺；若再把无契约 skill 全塞主上下文，你是在主动制造上下文压力。

一句话收束：**技能内容决定上限，组织与调用决定你能不能在长程里把上限用出来。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
