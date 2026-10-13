---
month: 202610
week: 2
date: 2026-10-13
type: 论文解读
slug: SWE-Duel
---

# 静态榜会漏进训练又被刷穿：让 agent 互出题互修，难度跟着作者一起涨

你有没有过这种体验：SWE-bench 一类仓库级榜单刚发榜时还很难，过几个月前沿模型就能复现 gold patch，主办方干脆不报分了；你换成「训练截止后的新 issue」续命，题目更新了，**难度天花板却仍卡在人类日常开发的分布上**。下一波更强的 agent 迟早又会饱和——**有没有一种评测，题目与 oracle 都在评测时现场生成，而且会随着参赛者变强而变难？**

新加坡管理大学 Wei Minn、Zhensu Sun、Lwin Khin Shar 与 David Lo 这篇 **SWE-Duel**（arXiv:2610.12569）把基准做成 Red–Blue 对战场：Red 在真实仓库里跨三个上下文隔离会话写新功能、埋隐蔽 bug、写密封触发测试，并通过盲修自检；Blue 在不知情的情况下审 PR，只有三类测试全过才算赢。看完我的感受：这是把 [GameArena](../../202609/week4/20260928_GameArena_论文解读_主观裁判太吵静态榜又饱和_用胜负当地面真值的对战评测台.md) 式「用胜负当地面真值」搬到 coding agent，并用可执行门禁把主观裁判彻底拿掉。

## 核心摘要

SWE-Duel 的目标是 **self-scaling**：任务与 oracle 在评测时由参赛者写出，事先无可泄漏内容；难度由参赛 agent 的出题能力决定，而不是发榜时锁死。每个回合里，Red 在仓库 $R$ 上经 Session A（实现功能 + 功能测试）、Session B（嵌入 bug + 密封 bug 测试）、Session C（盲修自检）三关，PR 只有在「功能测试与仓库回归通过、恰好 bug 测试失败，且同 agent 新会话能盲修」时才入库。Blue 只看到公开部分 $\mathrm{pub}(C)=(R,s,d,T_f)$，不知道 bug 是否存在；修订 $d'$ 必须同时通过已有测试 $T_e$、功能测试 $T_f$、密封 bug 测试 $T_b$ 才算 Blue 胜，否则 Red 胜。11 个 model–harness 组合在 15 个仓库、5 种语言上打满 **1,500** 个回合：GPT-5.5 + Codex CLI 以最强求解者居首（缺陷修复约 **98.3%**，竞赛 Blue 胜率 **98.7%**）；Claude Opus 4.8 + mini-swe-agent 以最强出题者居次（Red 胜率 **27.3%**）。同一模型族梯子实验里，最新版本出的题让固定求解者失败 **42.9%**，两个旧版合计仅 **4.0%**（1/25），给出难度随出题能力上升的初步证据。

## 论文信息

- **标题**：SWE-Duel: Towards a Self-Scaling Arena for Coding Agents
- **作者**：Wei Minn, Zhensu Sun（通讯）, Lwin Khin Shar, David Lo
- **机构**：Singapore Management University（School of Computing and Information Systems）
- **站点 / 工具**：https://sweduel.github.io/ · https://sweduel.github.io/SWE-Duel
- **链接**：https://arxiv.org/abs/2610.12569 （2026-10-08；HF Daily Papers 2026-10-12；ICLR 2027 under review）

---

## 🎯 为什么这件事值得写

静态 coding 基准有两条死路：内容泄漏进训练语料，以及固定题集被更强 agent 刷穿。持续更新的 SWE-bench-Live / SWE-rebench 能抗时间泄漏，但难度仍绑定人类 issue 分布；BugPilot 提前收集 agent 无意引入的回归，SwingArena 让提交者与写测试的 reviewer 对打，却仍依赖固定人类 issue 池或事先固定的任务集。

| 类型 | 任务从哪来 | 难度谁定 | 缺口 |
| --- | --- | --- | --- |
| 静态 SWE 榜 | 公开 issue / gold | 发榜时固定 | 泄漏 + 饱和 |
| 持续采集新 issue | 人类开发历史 | 仍人类分布 | 更新≠变难 |
| BugPilot / SwingArena | 部分模型生成 | 仍有固定锚 | 可见失败测试或人类 gold |
| SWE-Duel | 参赛者现场写 | 参赛者出题能力 | 需要自检门禁防无解题 |

和 [SchrodingerRepo](../../202609/week4/20260924_SchrodingerRepo_论文解读_SWE榜分可能记住了仓库长相_评测时再坍缩出陌生同构仓.md) 担心「记住仓库长相」同一焦虑：SWE-Duel 直接让评测时再长出新功能与新 bug。

## 🏗️ 机制：三会话出题 + 密封测试 + 硬合取计分

任务形式化为 $C=(R,s,d,T_f,T_b)$。Blue 只拿公开包，密封 $T_b$ 用于判定。胜负是硬合取：

$$
S_{\mathrm{Blue}}=S_e S_f S_b,\qquad S_{\mathrm{Red}}=1-S_{\mathrm{Blue}}
$$

没有「差不多修好」的部分分。

**Red 三关。**

1. **Session A**：自选模块实现新功能与 $T_f$；门禁要求 $T_e$、$T_f$ 通过，且 diff / 测试达最小规模。
2. **Session B**：在功能上埋 bug 并写 $T_b$；门禁要求带 bug 的 diff 仍过 $T_e$/$T_f$（Blue 看不到失败测试），且 $T_b$ 在无 bug 功能上过、在带 bug 版本上失败。
3. **Session C**：同 participant 新会话只拿公开包盲修；必须三套测试全过才入库，排除无解题，也意味着难度不会超过作者自己的找 bug 能力。

每个仓库请求 $N$ 个 slot，每 slot 最多五次尝试；耗尽则留空，对局时记为 Blue 自动胜——**造不出题也是被测能力的一部分。** 参赛者是 $(m,h)$ 对，同一模型换 harness 算两个 participant，各有自己的任务库。

**Tournament。** 先 round-robin；新加入者只需对 $\lceil\log_2 M\rceil+1$ 个在位者打，复用旧任务库与旧结果，Gemini 3.1 Pro 晚加入时五场就能落位。

## 🧪 关键证据

**主赛（Table 1，Bradley–Terry）。** 11 个 participant、15 仓、5 语言、约 50 场、1,500 回合。

| 排名 | Participant | Red 胜率 | Blue 胜率 | BT |
| --- | --- | --- | --- | --- |
| 1 | GPT-5.5 [Codex CLI] | 11.3% | 98.7% | +2.82 |
| 2 | Claude Opus 4.8 [mini-swe-agent] | 27.3% | 88.7% | +2.46 |
| 3 | GPT-5.5 [mini-swe-agent] | 10.4% | 97.0% | +2.13 |

两者登顶路径完全不同：GPT-5.5 是修 bug 机器；Opus 4.8 + mini-swe-agent 是埋更难找的 bug。同模型换 harness 也会大幅改排名——Opus 4.8 配 Claude Code 掉到中下游，说明 **测的是 model×harness，不是裸模型。**

**全场求解。** 1,500 回合 Blue 成功 87.9%；1,219 个 contested 回合里 85.1%（1,037 干净修复，158 漏掉 latent bug，24 次弄坏功能或回归）。按语言：Java 89.2% > Go 85.7% > C/C++ 83.9% > Python 83.5% > Node 82.9%。按自报 bug 类：逻辑错误最难修（81.9%），输入校验最易（95.5%）。

**难度随作者变强（Table 2）。** 同一家族 GLM-5.1 / 5.2 / 5.3 做出题梯子，固定参考求解者失败率：旧两版合计 **1/25=4.0%**，最新版 **6/14=42.9%**（趋势检验 p=.015）。这是 self-scaling 最直接的证据。

**泼冷水。** Session C 自检把门把难度钉在作者能力上限；出题失败给对手自动分，弱出题者会被放大惩罚；主赛成本高（多会话 × Docker × 多模型）；BT 排名对自动胜与 harness 选择敏感。

## 🔬 最有意思的部分

1. **bug 必须藏在新功能里。** Blue 不能靠回滚消 bug，否则功能测试也会挂——这比「在旧代码里种突变」更贴近真实 PR review。
2. **可见测试全绿，才叫隐蔽。** 和 BugPilot「弄坏已有测试才收录」相反：SWE-Duel 要求可见测试仍过，Blue 只能靠读代码找。
3. **出题与求解是两条能力轴。** 榜一修得最猛，榜二埋得最阴；只报总分会抹掉这种剖面，和 [CategoryAwareSWE](../../202609/week4/20260922_CategoryAwareSWE_论文解读_总分涨着一类却在掉_按类别训专家再蒸馏回来.md) 强调切片同一精神。
4. **晚加入者可对数场次落位。** 不必重打整张 round-robin，这对活着的 arena 很关键。
5. **和 GameArena 对照。** [GameArena](../../202609/week4/20260928_GameArena_论文解读_主观裁判太吵静态榜又饱和_用胜负当地面真值的对战评测台.md) 用对战胜负代替吵闹主观裁判；SWE-Duel 进一步用三类确定性测试当裁判，主观偏好彻底出局。

## 🤔 我的判断

定位：**面向 coding agent 的自扩展对战评测基础设施**，贡献在协议与门禁，而不只是又一张静态榜。

亮点三条：

1. 评测时生成任务 + 密封 oracle + 盲修自检，同时对抗泄漏、饱和与无解题；
2. 1,500 回合公开了 model×harness 的「出题 / 求解」双轴剖面，而不只是一个总分；
3. 模型族梯子实验给出难度随作者能力上升的可量化证据，并开源 arena 工具包便于续赛。

问题与局限：难度受作者自检能力封顶；自动胜机制放大弱出题者劣势；成本与工程复杂度高；仓库与语言集合仍是人工挑选的 15 个仓。

对工程的直接启发：**(1)** 做 agent 评测时同时报「造题成功率」和「盲修成功率」，别只报修 bug；**(2)** 新功能 + 隐蔽回归 + 密封测试，是比「破坏已有 CI」更接近真实 review 的构造；**(3)** 同一模型务必交叉至少两个 harness，再谈模型强弱；**(4)** 活榜要用可对数扩展的加赛协议，别每次全量重打；**(5)** 可与 [WideSWE](../../202609/week5/20260930_WideSWE_论文解读_单仓绿了不算完_跨仓协调才是coding_agent真考场.md) 的跨仓设定组合，把 self-scaling 从单仓对战扩到多仓协调。

一句话收束：**SWE-Duel 证明，coding 基准不必在发榜日锁死——让 agent 在真实仓库里互出隐蔽题、用密封测试决胜负，难度才能跟着作者能力一起涨。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
