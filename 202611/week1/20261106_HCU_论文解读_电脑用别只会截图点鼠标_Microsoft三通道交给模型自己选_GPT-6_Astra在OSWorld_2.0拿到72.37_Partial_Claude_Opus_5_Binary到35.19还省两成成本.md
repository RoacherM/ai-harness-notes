---
month: 202611
week: 1
date: 2026-11-06
type: 论文解读
slug: HCU
---

# 电脑用别只会截图点鼠标：Microsoft 把程序、语义、视觉三条通道交给模型自己选；GPT-6 Astra 在 OSWorld 2.0 拿到 72.37% Partial，Claude Opus 5 Binary 到 35.19% 还省两成成本；OSWorld 1.0 上四模型平均多 6.48 分、少 17% Steps、成本降 30%

你有没有过这种体验：让 agent 在桌面上改一张表，它先截一张图，再猜单元格在哪儿，鼠标点错一格就整段重来；明明可以用公式、CLI 或无树直接改，它却被「像人一样看屏幕」这件事捆死了。

这就是 **computer-use interface** 的分野。主流循环把「截图 + 键鼠」当成先验，把桌面里早就有的程序化接口和 accessibility（AX）语义通道留在黑暗里。Microsoft 这篇 *Hybrid Computer Use* 的核心问题是：**如果把程序、语义、视觉三条通道同时暴露给同一个 general agent，让模型自己选和组合，成功率和效率会不会一起动？** 看完我的感受是：**最该拿走的不是又一个榜上数字，而是「跨通道动作守卫 + AX 批量动作带 diff」这一套接口契约；通道选错可以靠 skill 纠正，状态过期却只能靠守卫当场挡住。**

## 核心摘要

HCU 通过 MCP 把两条桌面工具加进 agent 自带的程序化工具：语义通道读 AX 控件树、按引用地址批量动作并可选 wait，动作后默认回 AX diff；视觉通道截图与像素级键鼠输入。跨通道动作守卫在每次输入前对照 live 状态重检目标和布局，避免一条通道的动作让另一条通道手里的证据失效。配套 HCU skill 教模型何时选哪条通道、如何遵守调用契约。OSWorld 2.0 上，HCU 在对照系统里拿到最高 Partial 与 Binary：GPT-6 Astra 72.37% Partial（相对匹配的 CUA Driver +3.55，Binary +0.90，工具调用少 29.81%，成本几乎持平）；Claude Opus 5 Partial 相对官方榜 +3.75、Binary +3.76 到 35.19%，Cost 低 20.26%。OSWorld 1.0 上相对 CUA Driver，四条骨干平均 Score +6.48、Steps −17%、Cost −30%，GPT-5.4 在主设置到 70.13% Score；消融里 HCU 相对 CUA Driver +3.29 Score、Steps −29.80%、输入 token −56.85%、Cost −55.36%，其中 AX 贡献最大。Skill 把 AX/GUI 因参数畸形被拒的比例从 17.37% 压到 1.36%。同一套接口还迁到 WeaveBench（53.25 vs 46.37）、Windows Agent Arena（80.86 vs 71.89），Cost 大约减半。价值定位是：不改模型、只改 agent–computer 接口，让模型按任务拆分观察与执行通道。

## 论文信息

- **标题**：Hybrid Computer Use: A Unified Programmatic, Semantic, and Visual Interface for General Agents
- **作者**：Haoqing Wang、Xiaoyi Zhang、Sicheng Fan、Zhexin Zhang、Xinjie Zhang、Jiahao Li、Bin Li、Yan Lu
- **机构**：Microsoft
- **链接**：https://arxiv.org/abs/2610.41432 （约 2026-10-30 公开；HF Daily Papers 2026-11-05；GitHub https://github.com/microsoft/Hybrid-Computer-Use ；项目页 https://microsoft.github.io/Hybrid-Computer-Use/ ；2026-11-06 HF Daily API 尚未开放，本稿按 11-05 日榜条目归档）

---

## 🎯 为什么这件事值得写

计算机使用这条线最近同时挤着「纯 GUI 循环」「程序 + GUI 混合」「训练侧 harness 进化」几股力量。HCU 站在 **训练无关、接口优先** 这一侧，有三点特别值得单独记：

1. **它把 AX 从「给 GUI 找坐标的辅助」抬成一等公民。** 早期 OSWorld 的 set-of-marks 用 AX 标元素，最后仍映射成坐标点击；HCU 直接按引用地址做动作，并把 wait 与 post-action AX diff 塞进同一次调用。
2. **它正面处理跨通道状态过期。** 三条通道动的是同一张桌面，一条通道的写入会让另一条通道手里的截图或控件引用失效；守卫在执行前重检，这比事后靠模型「再看一眼」更硬。
3. **它把通道选择交给模型，再用 skill 约束调用形状。** 四个骨干拿到同一接口后分工完全不同：GPT 桌面调用里 AX 约占 80%，Claude 反而 94% 走 GUI 且更爱批量动作。接口设计必须容忍这种异构，而不是假定「最优通道」唯一。

对我们关心的 harness / MCP / skill 线，这篇几乎是一份可直接抄的「桌面工具契约」样本。

## 🏗️ 机制：三通道 + 守卫 + skill

### 三条通道

| 通道 | 暴露方式 | 典型用途 |
| --- | --- | --- |
| 程序化 | agent 自带的 code / shell / 文件工具 | 文件、数据、脚本、CLI |
| 语义（AX） | MCP 桌面工具：紧凑观测、引用地址、批量动作 + wait、默认 AX diff | 命名控件、可组合的 UI 操作 |
| 视觉（GUI） | MCP：截图与键鼠像素输入 | AX 不够用时的视觉证据与像素操作 |

### AX 设计要点

- **紧凑观测**：不把整棵巨大控件树原样灌进上下文。
- **同调用批量动作 + 可选 wait**：少一轮「做完再问效果」。
- **默认反馈是 AX diff**：skill 打开后，87% 的 AX 动作调用省略 `observe` 字段，直接吃默认 diff；没有 skill 时这一比例只有 0.2%。
- **跨通道守卫**：切换或混用通道前，对照 live 状态重检目标与布局。

### HCU skill

Skill 不替代模型决策，而是说明何时用哪条通道、参数怎么填、失败怎么收。效果主要体现在调用可靠性：畸形参数拒收 17.37% → 1.36%；同一工具反复换参还被拒的 task-run 比例 44.69% → 0.37%；每任务 GUI 输入动作从 4.96 降到 1.97，Steps 再少 14.55%，Score +1.59。

### 实验口径

| 项 | 设置 |
| --- | --- |
| 主基准 | OSWorld 2.0（Partial / Binary）、OSWorld 1.0（Score） |
| 迁移 | WeaveBench、MacAgentBench 子集、Windows Agent Arena |
| 基线 | CUA Driver（同三通道但策略不同）、官方榜、StateAct 等 |
| 步数上限 | 主实验 500 step；消融 100 step |
| 消融骨干 | GPT-5.4；跨模型对照另含 Claude Opus 5.5、Gemini 3.8 Flash、Grok 4.7 |

## 🧪 关键证据

### OSWorld 2.0（Table 1 摘录）

| 系统 | 模型 | Partial (%) | Binary (%) | Cost ($) |
| --- | --- | --- | --- | --- |
| Computer-use loop（官方） | Claude Opus 5 | 68.31 | 31.43 | 28.43 |
| StateAct | Claude Opus 4.8 | 61.60 | 26.90 | ~7.80 |
| CUA Driver (Codex) | GPT-6 Astra | 68.82 | 31.50 | 10.05 |
| **HCU (Codex)** | **GPT-6 Astra** | **72.37** | **32.40** | **10.03** |
| **HCU (Claude Code)** | **Claude Opus 5** | **72.06** | **35.19** | **22.67** |

读法：HCU 在全量对照里同时占住最高 Partial 与最高 Binary；与 GPT-6 Astra 的 CUA Driver 比，Partial +3.55、工具调用 −29.81%、Cost 几乎不变。和 Claude 官方榜比，Binary 到 35.19% 且 Cost 低约两成。StateAct 更便宜（约 $7.80）但 Partial 只有 61.60%，和 HCU+GPT-6 Astra（$10.03 / 72.37%）一起落在 Pareto 前沿两侧。

### OSWorld 1.0：四模型 vs CUA Driver

| 模型 | CUA Driver Score | HCU Score | Δ (pp) | Steps | Input | Cost |
| --- | --- | --- | --- | --- | --- | --- |
| GPT-5.4 | 62.30 | 65.59 | +3.29 | −30% | −57% | −55% |
| Claude Opus 5.5 | 68.68 | 71.62 | +2.94 | −1% | −32% | −8% |
| Gemini 3.8 Flash | 54.67 | 67.57 | +12.90 | −17% | −39% | −44% |
| Grok 4.7 | 67.90 | 74.71 | +6.81 | −22% | −18% | −13% |

摘要里的「平均 +6.48 Score、−17% Steps、−30% Cost」以及 GPT-5.4 主设置 70.13% Score，对应更完整的主实验口径；上表是 100-step 消融同设定下的细表。跨平台：WeaveBench 53.25 vs 46.37，WAA 80.86 vs 71.89，MacAgentBench 子集 25/28 vs 24/28，Cost 低 46–51%。

### 消融（GPT-5.4，OSWorld 1.0）

| 配置 | Score | Steps | Input (M) | Cost ($) |
| --- | --- | --- | --- | --- |
| HCU | 65.59 | 28.60 | 0.941 | 2.52 |
| w/o GUI | 66.08 | 32.16 | 0.909 | 2.45 |
| w/o AX | 62.79 | 33.28 | 1.051 | 2.82 |
| w/o skill | 64.00 | 33.47 | 0.962 | 2.57 |
| CUA Driver | 62.30 | 40.74 | 2.181 | 5.65 |

相对 CUA Driver：+3.29 Score、Steps −29.80%、输入 token −56.85%、Time −29.85%、Cost −55.36%。去掉 AX 损失最大（Score −2.80 相对完整 HCU）；去掉 GUI 分数几乎不动甚至略高，但 Steps +12.45%。只留 AX + 程序化时，仍比 CUA Driver 高 3.79 Score，同时更省资源。

## 🔬 最有意思的部分

1. **观察和执行可以走不同通道。** GPT / Gemini / Grok 在 29–31%「用过 GUI」的任务里，全部 GUI 调用其实只是截图，真正动手走 AX。截图确认 AX 反馈里看不到的值，再继续用 AX 执行，这种分工只有「通道选择留给模型」时才会自然出现。
2. **Score 增益几乎都来自「真的用了桌面工具」的任务。** 没调用 AX/GUI 的任务组，对总分差的贡献只有 −0.7 到 +0.1。接口红利不是均匀撒在所有题上的。
3. **Claude 的 Steps 几乎不变，不等于没变高效。** 两边都用桌面工具的任务上 HCU 少 19–40% Steps；但 Claude 有更多「只有 HCU 才去碰桌面工具」的任务，Steps 上升、Score 多 7.6 点，两边对冲后总 Steps 看起来持平。只看总步数会误判。
4. **苦涩教训式的读法。** 作者写得很直：以人类操作方式为先验的接口，会同时牺牲性能和效率；给了 AX 之后，四个模型里有三个把大部分桌面调用交给 AX。电脑上本来就有的能力，不必再训练一遍模型。

## 🤔 我的判断

**定位：一篇接口设计论文，而不是新模型或新训练配方；证据覆盖 OSWorld 1.0/2.0 与多平台迁移，对 harness 工程的直接产品是三通道 MCP 工具契约、跨通道守卫和一份可降低畸形调用的 skill。**

亮点：

1. **把 AX 做成执行中心而不是标注辅助**，批量动作、wait、diff 同调用返回。
2. **跨通道守卫**正面处理「状态过期」这个混合接口的硬伤。
3. **同一接口、不同模型、不同分工** 的行为分析，说明「统一接口」不等于「统一策略」。

问题与局限：

- **对照不完全同 harness。** OSWorld 2.0 主表混有官方榜、StateAct、厂商结果；最干净的是与匹配 CUA Driver 的成对比较。
- **消融步数上限（100）与主实验（500）不同**，GPT-5.4 的 65.59 与摘要 70.13 不要混读。
- **成本是标价估算**，跨模型、跨 harness 时输出 token 结构差很大（Claude Code 路径 Out/1k 更高）。
- **MacAgentBench 只报了子集 25 vs 24 / 28**，样本小，只能当迁移信号。
- **Skill 依赖模型肯读、肯遵守**；换更弱的模型或更吵的系统提示，拒收率曲线未必复现。

对工程的直接启发：

- **桌面 agent 默认同时挂程序化、AX、GUI 三条 MCP 工具**，不要只给截图键鼠。
- **动作前做 live 状态重检**，尤其在通道切换点；这比事后「再截一张」便宜且硬。
- **AX 调用默认返回 diff，而不是整树或整屏**；把验证嵌进工具契约。
- **skill 写成通道选择与参数契约**，专门打畸形参数和重复拒收。
- **评测时按「是否调用了桌面工具」切分增益**，避免被总 Steps 平均数骗到。

一句话收束：**HCU 的主张是，让模型像程序员一样用电脑，而不是逼它永远扮演一个只会盯屏幕的人。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
