---
month: 202610
week: 3
date: 2026-10-15
type: 论文解读
slug: FailConf
---

# Agent 能学会预言自己失败吗：用最终成败校准每步置信度

你有没有过这种体验：长程 agent 跑到第几十步，你想在「继续 / 复核 / 交给人」之间做闸——于是让模型口头报一个 `<conf>72</conf>`。没额外训练时，小模型的口头置信度几乎随机；用 RL 去校准的工作，又多半对着**当前步答对没有**或**单题答案对不对**打分。长程任务合法路径很多，局部正确性本身就说不清。**真正要预测的，难道不该是「这一整次 rollout 最终会不会成功」吗？**

SAP 的 Ashutosh Hathidara 这篇 **Can an Agent Learn to Predict Its Own Failure?**（arXiv:2610.18614）把校准目标改成**结果锚定（outcome-grounded）**：在 GRPO 的一组同任务 rollout 上，用最终成败 $Y$ 当标签，用 proper scoring rule 给每一步的口头置信度打分。看完我的感受：训练确实能把弱 agent 的置信度变成可用的**早期分流信号**——但增益主要是「更早认出哪些任务会挂」，不是「同任务里分清哪次尝试会赢」。

## 核心摘要

作者把逐步置信度目标写成 $c_t \approx P(Y=1\mid h_t)$，即给定当前历史，本局最终成功的概率；置信度从 `<conf>NN</conf>` 解析（缺省中性 0.5）。离线 RL 在 Qwen2.5-7B agent 的 ALFWorld / TextWorld / WebShop 录制轨迹上，逐步置信对最终结局的 pooled AUROC 从近随机升到 **0.68–0.85**，最自信的 top-10% 步骤成功率约为平均的 **∼4.3×**。在线 RL 于 live ALFWorld：四种校准奖励都抬高早期窗口（$k\le 2$）AUROC（如 Brier+dispersion **+0.101**），ECE 从 **0.42** 降到 **0.05–0.12**，选择性执行优于未训练与 format-only 对照，且**不牺牲**任务成功率（约 0.19–0.20）。作者另给出拆开深度、任务难度、尝试级预见的评估协议：增益主要来自 **between-task**（第一步就认出难任务），ALFWorld 上 within-task 尝试级预见几乎不动；TextWorld 早期窗口则出现 within-task 增益。价值定位：把「口头置信」从单步正确性校准，改造成 agent 控制用的**结局预见闸**。

## 论文信息

- **标题**：Can an Agent Learn to Predict Its Own Failure? Outcome-Grounded Confidence Calibration in LLM Agents
- **作者**：Ashutosh Hathidara
- **机构**：SAP
- **链接**：https://arxiv.org/abs/2610.18614 （2026-10-13；HF Daily Papers 约 2026-10-14/15）

---

## 🎯 为什么这件事值得写

Agent 控制栈需要的不是「这步像不像对」，而是「继续这条轨迹值不值得」。这和 [LLM-as-Judge](../../202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 的闭环焦虑同一层：闸要有可校准的信号。同日 [ActWrong](./20261015_ActWrong_论文解读_读对却做错_System_One在agent循环里的决策失败.md) 显示 System One 的原生置信校准参差（ECE 2.6–26.6%）；本文则问：**acting LLM 自己的口头置信，能不能被训成结局预测器？**

Concurrent 工作（RLCR、CAR/M-SCR、CAPO）多对准同步正确性或步级正确；长程 agent 合法动作不唯一，局部正确难定义——所以作者坚持用最终 $Y$。

| 估计量 | 固定什么 | 测什么 |
| --- | --- | --- |
| Pooled AUROC | 无 | 混难度+深度+尝试 |
| Between-task | 每任务均值 | 任务难度排序 |
| Within-task | 同任务成败尝试 | 尝试级预见 |
| Depth-matched within-task | 同任务同步索引 | 纯预见 |

## 🏗️ 机制：GRPO + proper scoring 逐步奖励

**策略。** $(a_t,c_t)\sim\pi_\theta(\cdot\mid h_t)$，整条轨迹得二元 $Y$。

**逐步奖励。** $r = w_{\mathrm{cal}}\,\mathrm{cal}(c,Y) + w_\varphi\,\varphi(c) + w_{\mathrm{fmt}}\,e$，默认 $\mathrm{cal}$ 为 Brier $[1-(c-Y)^2]$；可选 spread/dispersion 把置信度从端点拉开；format 奖励保证标签可解析。

**组相对优势。** 同任务 $G$ 次尝试的 $Y_i$ 零成本当标签：
$$A_{s,i,t}=(Y_{s,i}-\bar Y_s)+(r_{s,i,t}-\bar r_s)$$
同时奖励「这局赢了」与「这步置信报得准」。

**训练制度。** 离线：单条录制轨迹；在线：live agent 同环境采样。对照含未训练、format-only、token entropy / logprob 白盒信号、仅深度基线（$-t$）。

## 🧪 关键证据

**离线（Table 1）。** Pooled AUROC：ALFWorld 0.517→**0.681**，TextWorld 0.578→**0.722**，WebShop 0.462→**0.850**；top-10% 尾部成功率提升到约 **4.3×** 基率。校准项有无比函数形式更关键（Table 2：spherical/Brier/ranking 等 AUROC 0.820–0.882 重叠；仅任务奖励只到 0.559）。优势估计器：同设定下 RLOO 可到 0.801/0.876，GRPO 0.613/0.676，冷启动 value critic 的 PPO 学不会可解析格式（0.500）。

**在线 ALFWorld（Table 4，Qwen2.5-7B-Instruct）。** 早期窗口 pooled AUROC：未训练 **0.689**；Brier+φ **0.761**（Δ+0.072）、Balanced Brier **0.765**（+0.076）、Brier+dispersion **0.790**（+0.101）、Balanced+dispersion **0.737**（+0.048）；format-only 仅 +0.019（CI 含 0）。ECE（全步）0.419→**0.053–0.115**，Brier 0.250→**0.044–0.054**。Top-10% 早期成功：0.493→**0.56–0.66**。任务成功率保持 **0.189–0.203** vs 未训练 0.201。

**增益拆解（Table 5 / §7）。** Between-task 早期 AUROC 升（如 Brier+dispersion Δ **+0.072**）；within-task 配对 Δ 多为负或 CI 含 0——预注册的 within-task 准则未达成。Step-0 pooled AUROC 从 **0.562** 升到 **0.642–0.700**，但 within-task step-0 仍在机会水平：模型在**还没收到环境反馈前**就按任务难度分流，而不是给第一次动作打分。

**迁移与规模。** TextWorld 在线 Brier+φ：成功率 0.146→0.262；within-task AUROC step-0 **+0.265**、前三步 **+0.070**。Qwen2.5-14B 同协议 pooled matched-emission AUROC 0.697→0.729（Δ+0.032）。

**泼冷水。** 证据最强的是 between-task；同任务尝试级预见在 ALFWorld 未显著；高置信 bin 仍可能过度自信，控制器宜用「高置信带」而非只信最高分；离线区间按步 bootstrap 会低估同轨迹相关；部分 within-task 子集簇很少、CI 很宽。

## 🔬 最有意思的部分

1. **深度会冒充预见。** 仅用步深+输出长度就能在全步 pooled 上打平甚至超过训练后置信；真正要看早期窗口与 depth-matched——未训练深度基线在早期塌到机会水平。
2. **训练前后白盒信号易位。** 训前 token 熵常强过口头置信；训后策略变尖、熵信号崩，口头置信反超。
3. **Format-only 几乎不动分流。** 证明增益来自校准项，不是「学会写 `<conf>`」。
4. **第一步就能分流任务。** 均匀结局组会把整条轨迹的置信推向共享结局，于是 step 0 也被带着学——这是机制，也是 within-task 难涨的原因。
5. **和 ActWrong / Jev 互补。** ActWrong 用原生分布做级联；本文教 acting 模型用口头置信做 triage——两者都是「不确定就升级」栈的零件。

## 🤔 我的判断

定位：**agent 控制用的结局锚定校准方法 + 评估协议**，不是再刷环境成功率。

亮点三条：

1. 把校准标签从「步对/题对」改成「局成」，对准长程控制真实需求；
2. 四估计量拆开深度/难度/尝试混淆，避免 pooled AUROC 自嗨；
3. 在线结果：ECE 大降、早期分流升、任务成功率不掉。

问题与局限：ALFWorld 上 within-task 预见弱；主要在 7B（14B 样本少）；环境以文本仿真为主；高置信尾部仍需谨慎。

对工程的直接启发：**(1)** 做 abstain/handoff 闸时，用**最终成败**训置信，而不是用局部步对；**(2)** 报表至少拆 early-window / between-task / within-task，别只报 pooled；**(3)** format 奖励不够，必须有 proper scoring 校准项；**(4)** 控制器用「高置信带」做选择性执行，别只信最顶端；**(5)** 若目标是「同任务多试几次里哪次会成」，当前配方不够——需要显式对比同任务成败尝试的信号。

一句话收束：**Agent 可以学会更早预言自己会在哪些任务上失败；但别把它误读成已经会给同一次任务的每一次尝试打分。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
