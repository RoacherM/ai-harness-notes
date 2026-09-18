---
month: 202609
week: 3
date: 2026-09-18
type: 论文解读
slug: Harness-Design
---

# 脚手架不是一整块：Zoom 这篇用 176 组对照证明，组件要按模型和预算配

你有没有过这种体验：同一模型换一套 coding harness，榜上差一截；但换完之后你也说不清，到底是 planning 帮忙、工具集帮忙，还是上下文压缩帮忙。行业里大家常把 Claude Code / Codex / OpenHands / SWE-Agent 当「整机」互比，Cao et al. 甚至报过 Opus 偏爱 OpenHands、Sonnet 偏爱 SWE-Agent——**整机对比把机制揉在一起，差分没法归因**。Zoom Video Communications（含实习）与 UMass / Emory / UNC Charlotte 这篇 *An Empirical Study of Harness Design for Coding Agents*（arXiv:2609.20804）做的是相反的事：固定 ReAct 执行环，只动 **planning / action space / context management** 三块，在 Nemotron-3（30B/120B/550B）与 Mistral-Medium-3.5 上跑 SWE-Bench Verified + Terminal-Bench 2.1，共 **176** 个 matched settings。看完我的感受是：这不是「又一个更好 harness」，而是给 Byron 栈里 Pi / DSH / ACP 做 **model- & budget-aware 选型清单** 的测量基础设施。

## 核心摘要

作者自研轻量 harness（LangGraph + Harbor 容器评测），默认工具集含读写/搜索/`bash`/`web_fetch`，外加 `update_plan` 与（T2/T4 下）`recall_event`。上下文管理拆成三个机制再组五档：M1 规则 elision、M2 外存 + `recall_event`、M3 同模 LLM summarization；T0 无管理（超窗即死），T1=M1，T2=M1+M2，T3=M3，T4 在软阈 \(B_1\) 先 elision、硬阈 \(B_2\) 再 summarization（preamble + 近期窗口原文保留）。在 32k/64k/96k/128k 四档窗预算上扫 T0–T4，并在 T4/128k 上单独关掉 planning 或换成 bash-only。主结论四条：(1) **上下文管理的价值随窗收紧而升**，收益主要来自消灭 overflow 早死——SWE 上 managed 相对 T0 的 SR 差距从 32k 的 **35.7** 点收到 128k 的 **2.7** 点；(2) **T4 整体效率最强**：精度与 T1–T3 接近，但用廉价早 elision 少打贵的 summarization，多数 model–benchmark 面板上成本最低；而可恢复的 M2 **几乎不被调用**，相对纯 elision 无稳定精度增益；(3) **Planning 随能力换角色**：弱模（30B）靠它撑到第一次 edit（SWE +11.6 点），强模（550B / Mistral）主要靠它砍掉冗余 post-edit verification、成本约降 **30%**，精度几乎不动甚至略降；(4) **预定义工具脚手架弱 bash 模，bash-only 便宜且对强模更友好**——550B 在 bash-only 上 SWE/TB 分别 +3.6 / +5.6 点、成本 −53% / −30%。轨迹分析把三条效应钉死：上下文管理**拉长轨迹但不改行为分布**；planning 改的是**在哪停**；action space 改的是**写代码的粒度**。

## 论文信息

- **标题**：An Empirical Study of Harness Design for Coding Agents
- **作者**：Run-Ze Fan, Zihao Zhang, Simin Ma, Yebowen Hu, Shouju Wang, Kaiqiang Song, Fei Liu, Hamed Zamani, Xiaoyang Wang（部分工作于 Zoom 实习期间完成；通讯邮箱含 `Xiaoyang.W@zoom.us`）
- **机构**：Zoom Video Communications · UMass Amherst · Emory University · UNC Charlotte
- **链接**：https://arxiv.org/abs/2609.20804 （2026-09-17 提交，43 页）· HF https://huggingface.co/papers/2609.20804

---

## 🎯 为什么「整机 harness 榜」不够用

Coding harness 是把 LLM 变成 agent 的软件层：控制环、工具接口、上下文策略。已有产品与框架（Claude Code、Codex、OpenCode、OpenHands）和长记忆 / 任务图类工作，多半优化的是**捆绑设计**。整机对比能告诉你「这对模型-harness 更好」，但不能告诉你：换 planning 还是换工具，还是换压缩策略？作者的研究问题很直接——

> 组件是一般有用，还是其效用依赖模型能力、任务类型与资源预算？

他们刻意只动三块互补需求：planning 维持进度、action space 把意图变成可执行操作、context management 在有限窗口里留住有用历史；权限门、post-edit diagnostics（ruff/pyflakes）、stuck detection（同调用连败提醒/早停）全部**锁死**，充当公共执行底盘。这和 Byron 做 DSH / ACP 时「先固定评测环，再扫策略旋钮」是同构方法论。

## 🏗️ 机制：固定环，扫三轴

### Planning

开启时：系统协议 + 首轮提醒先出 plan；`update_plan` 维护持久 todo；后续轮次把 plan **注入输入但不写入对话历史**。关掉时：指令、提醒、注入与工具一并移除。因此消融估计的是「**持久规划脚手架**」，不是「模型会不会在 CoT 里自己想步骤」。

### Action space

- **Full tools**：`read_file` / `write_file` / `edit_file` / `list_files` / `glob_files` / `grep_text` / `web_fetch` / `bash`（故意不做 web search，防 SWE 题面泄漏 PR）。
- **Bash-only**：去掉预定义文件/搜索/web，只留 `bash`；planning / recall 辅助工具仍按档保留。

重要诚实点：这是**整套接口干预**（工具可用性 + 接口说明 + 文件状态追踪 + 自动诊断），不是「工具个数」的纯因子。

### Context management（T0–T4）

| Tier | M1 elision | M2 recall | M3 summary |
| --- | --- | --- | --- |
| T0 | ✗ | ✗ | ✗ |
| T1 | ✓ | ✗ | ✗ |
| T2 | ✓ | ✓ | ✗ |
| T3 | ✗ | ✗ | ✓ |
| T4 | ✓ | ✓ | ✓ |

T4 算法要点：超 \(B_1\)（约 0.6× 可用窗）先把中间区臃肿 tool 输出换成 stub 并外存；仍超 \(B_2\)（约 0.85×）才对最旧中间事件做 summarization；preamble + ≥2 轮近期窗口原文保留。阈值与 soft/hard 阈值配置写进正文，可复现。

## 🧪 关键证据：176 设置不是装饰

**Setup 要点**：SGLang BF16、temperature 0、每任务 ≤300 step；OpenRouter 计价（2026-08）；McNemar + BH-FDR 0.05。SWE-Bench Verified 500 题；TB 2.1 共 89 题。

**上下文管理**：所有 managed tier 在任意预算下 **overflow = 0**；T0 的 overflow 随窗从 ~79%/61%（SWE/TB @32k）掉到 ~9%/12%（@128k），managed–T0 的 SR 差距同步收窄。T4 在八个 model–benchmark 面板上多数成本最低：早 elision 压住峰值上下文，减少 M3 调用。

**Recall 泼冷水**：64 个 T2/T4 设置里 **56.3%** 从未调用 `recall_event`；均调用从 32k 的 0.54 降到 128k 的 0.007；重用集中在最弱的 30B。T2 vs T1 跨 32 对照几乎打平（均差 −0.36 点）。**无损可恢复 ≠ 有用**——模型（尤其强模）不会去捞被 elide 的东西。

**Planning**：30B 关 planning 后 SWE 中位轨迹 40→5 轮，无 edit 终止 68.6%，Localize 卡死 58.4%；开 planning 分别降到 27.8% / 10.4%。550B / Mistral 则中位轨迹显著缩短，砍的是 Verify 相而非 Localize。

**Bash-only**：30B 离开预定义工具大量「发训练时见过、注册表里没有的 tool call」，TB 上 66% 轨迹因此早死；550B 反而用 shell 把多步捆成单次调用——re-patch 下降、create/replace 占比上升、成本大降。

## 🔬 最有意思的部分：三条「行为机制」对应三条局限

作者在 §4 用 LLM-judge 轨迹标注把数字翻译成可迁移诊断：

1. Context management → **治早死**（窗不够时）；窗够大时边际精度收益变模型依赖。
2. Planning → **弱模：撑到第一次 edit；强模：教它停**（少做无效 verification）。
3. Action space → **弱模：对齐可调用词表；强模：允许粗粒度组合动作**。

这对「默认全开」的工业 harness 是直接打脸：可恢复记忆、永远开 plan、永远给满工具——每一项都有反例。局限也很诚实：planning / action 只在 T4/128k 消融；每任务单次运行；TB 仅 89 题许多格子不显著；SWE 偏 Python；模型家族覆盖有限。

## 🤔 我的判断

定位：**harness 组件级的测量基础设施 + 条件化系统设计指南**，不是新 SOTA agent。亮点三：

1. 把「整机互比」拆成可归因三轴，并用 176 matched settings + 轨迹相分析闭环；
2. T4「规则 elision 在前、LLM summary 在后」是可直接抄进 Pi / DSH 的默认上下文策略；
3. 明确写出 **recall 机制经常是死代码**、**planning 对强模主要是省钱**——纠正了不少「越多脚手架越好」的默认。

对 Byron 栈的直接启发：

- **Pi / DSH**：上下文默认走 T4 思路；别急着上可召回外存，除非你有证据 agent 真的会 `recall`。
- **ACP / judge**：轨迹相标注（Localize / Edit / Verify）比单点 success 更能解释「为什么某旋钮有效」——和 Meta-Harness「完整轨迹文件系统」同一精神，只是这篇站在**组件测量**侧。
- **Wayne-Skills / CL**：planning 脚手架对弱后端是 accuracy scaffold，对强后端是 cost saver——skill 里「强制先写 plan」应按后端分档，不要一刀切。

一句话收束：**下一个 harness 不该追求「功能最多」，而该追求「在目标模型 × 任务类型 × 窗预算下，每个组件都在治自己该治的失败模式」。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
