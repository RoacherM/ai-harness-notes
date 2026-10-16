---
month: 202610
week: 3
date: 2026-10-16
type: 论文解读
slug: JevAgentHarness
---

# 孤立题答对不等于能控住 agent：决策模型要拟合阈值，才敢共享控制

你有没有过这种体验：把 Jev、OpenJev、StartLux 这类 System One 决策模型挂进 coding / 客服 agent，想省掉每步一次大模型调用——孤立工具选择题上它明明又准又便宜，一进真实循环却把任务搞砸：自信地 cancel 了本该先 ask 的订单，错一步后面全偏。**孤立准确率高，到底能不能翻译成「让它替 LLM 做选择题」？**

UIUC、Princeton 与 Google 这篇 **JevAgentHarness**（arXiv:2610.21412）把问题钉成三件套：用 **AgentDecisionBench** 从孤立题测到「跑着的 agent 由谁拍板」，用 **JevAgentHarness** 按决策点拟合置信阈值并只在够自信时接管，再用 agent 自己踩过的状态（含自己犯错后的状态）训出开源 **AgentDecision-4B**。看完我的感受：仓库里 [ActWrong](./20261015_ActWrong_论文解读_读对却做错_System_One在agent循环里的决策失败.md) 说快系统会「读对却做错」；这篇补上部署答案——**别信孤立榜，信任务级；别无阈值 REPLACE，信拟合 CASCADE；别用固定题集训，信 agent 轨迹上的 Jev 标签。**

## 核心摘要

多数 agent harness 每步都调大 LLM，但多数调用落点是多项选择题（下一工具、参数槽、是否停、是否反思）。决策模型（Jev 等）直接给各选项概率，一次记录里 Jev 调用只要 LLM 的 28% 时间与 2.6% 价格。作者定义八个决策点 × 五种接入模式（REPLACE / CASCADE / ASSIST / GUARD / TRIGGER），合 40 种委托；CASCADE 仅当置信度 $c(s)\ge\tau_d$ 时由决策模型拍板，否则回退 LLM。评测栈 **AgentDecisionBench** 三层：L1 孤立题、L2 录制轨迹逐步打分（不执行）、L3 真实控制跑任务；环境含公开榜与自建 **DecisionGym**（每态可知可接受下一步）。关键反转：孤立题最准的 StartLux-9B（BFCL 工具选择 92.2%）在 L3 只解出 **69.5%** 任务，而 BFCL 仅 89.3% 的 AgentDecision-4B 解出 **91.5%**，Jev 为 **82.0%**；无拟合阈值时让 Jev 包办每个工具选择会掉 **19.0** 点成功。部署上，按 agent 轨迹拟合阈值的工具/参数 cascade 在 500 题上切 **14.2%** 成本且零任务变坏；固定阈值 0.5 则掉 **21.5** 点。用 Jev 标签在 agent 访问状态上 SFT（再可选 GRPO）训出的 4B，在分布内可让 DecisionGym agent **贵 86.3%、快 2.5×**，ALFWorld 上自训 4B 让 agent **便宜 89.0%**，且不丢 ≥3 点成功。价值定位：把「决策模型进 harness」从路由器轶事，变成可测、可阈值、可自训的共享控制管线。

## 论文信息

- **标题**：JevAgentHarness: Benchmarking, Deploying and Training Decision Models That Share Control with LLMs
- **作者**：Kunlun Zhu, Tao Feng, Xu Han, Heng Wang, Ling Yang, Xiangru Tang, Jiaxuan You, Heng Ji
- **机构**：University of Illinois Urbana-Champaign · Princeton University · Google
- **仓库**：https://github.com/JevAgentHarness/JevAgentHarness
- **链接**：https://arxiv.org/abs/2610.21412 （2026-10-15）

---

## 🎯 为什么这件事值得写

Harness 成本的大头是「每步一次大模型」。路由器、小函数调用模型、快慢双模型都在挪活，但多在固定题或单一决策点上验收；agent 榜又只报整任务成功。决策模型一旦进循环，错会传到下一步，还会撞上「agent 自己犯错后的状态」——固定题集几乎没有。

这和 [JEV-as-a-Judge](../../202609/week4/20260924_JEV-as-a-Judge_论文解读_自信就收下不确定就升级_决策裁判三分钱顶住强模型.md) / [JustAskJev](../../202609/week4/20260928_JustAskJev_论文解读_对齐失败检测别为每条准则再解码_一次问多题零样本就能筛.md) 把 Jev 当快裁判同一家族，也和同周 [ActWrong](./20261015_ActWrong_论文解读_读对却做错_System_One在agent循环里的决策失败.md) 的「读对 ≠ 决策对」咬合：本文问的是**部署接口**——什么时候把选择题交给它、阈值怎么拟合、模型要不要在 agent 状态上重训。

| 测法 | 量什么 | 会漏什么 |
| --- | --- | --- |
| L1 孤立题 | 选项打分准不准 | 错误是否可逆、是否传播 |
| L2 录制步进 | 在真实历史点上准不准 | 答案不执行，状态不偏航 |
| L3 任务级控制 | 整任务成功 / 成本 | 需要可打分环境或 DecisionGym |

## 🏗️ 机制：八决策点 × 五模式 + 按点拟合阈值

**决策点。** 三类选动作（工具、参数、无工具时下一动作）；五类管循环（写记忆、读记忆、失败后反思、停止、路由到更便宜 LLM）。

**五种模式。** REPLACE 永远决策模型；CASCADE 够自信才接管；ASSIST 短名单给 LLM；GUARD 审 LLM 提案；TRIGGER 决定可选步骤（如反思）开不开。一篇委托 = 决策点 + 模式 + 阈值 $\tau_d$。目标：找最便宜、成功相对「纯 LLM」掉分不超过 3 点（$\tau^2$-bench 上 5 点）的委托集合。

**阈值规则。** 若置信近似正确率，值得移交当 $c(s)\ge q_L-\Delta C/V$（$q_L$ 为 LLM 正确率，$\Delta C$ 省下的调用价，$V$ 为错一次的任务价值损失）。实践上先在校准任务静默记决策模型输出，再按置信分桶估准确率拟合 $\tau_d$；工具点示例拟合到 1.00、参数点到 0.99 量级（文中部署例）。

**AgentDecision-4B。** 在 agent 访问状态上以 Jev 答案为标签做 SFT，再可选 GRPO（奖励含任务成功、少 LLM 调用、错选惩罚）。动机：现成决策模型几乎没见过「自己犯错后的状态」。

**两阶段流水线。** Stage 1 单独测每种委托好不好更便宜；通过的冻结。Stage 2 把通过的组合（文中最终是工具+参数 cascade + 反思 trigger）放到新任务上一起测。

## 🧪 关键证据

**Finding 1：孤立分会排错名。** Figure 4b（shifted DecisionGym）：L1 分差仅 5.1 点，L3 任务成功分差拉到 **57.5** 点。StartLux-9B 孤立最强却 L3 **69.5%**；AgentDecision-4B L3 **91.5%**，Jev **82.0%**，OpenJev-27B **69.5%**，Intern-Decision-4B 仅 **34.0%**。步进准确率与任务成功 Kendall 相关 **0.84**；缺口在 must-ask 状态（4B 67.4% vs Jev 58.3%）。

**Finding 2：拟合阈值护住成功。** 无阈值让 Jev 全包工具选择：−**19.0** 点；三工具 ASSIST 短名单：−**7.8** 点但省 53.7% 成本。拟合 CASCADE：Jev 接管 27.2% 工具选择与 20.8% 参数选择，500 题上省 **14.2%** 成本、零任务变坏。阈值钉死 0.5：−**21.5** 点。Flash-Lite 当 agent LLM 时拟合阈值会把工具选择全留给 LLM，cascade 省不了钱——说明阈值是 agent 相关的，不是普适魔法数字。

**Finding 3：部署省钱（Table 1 / Table 2）。** AgentDecision-4B 独占 GPU：DecisionGym **−86.3% 成本、2.5× 更快**；ALFWorld 自训 4B：**−89.0% 成本**且成功 90.7→95.0。4B 中位应答 **31 ms** vs Flash **809 ms**；500 新任务上 4B 做了 99.9% 工具选择与 75.0% 参数选择，每任务少约 4.5 次 LLM 调用。编码 agent 上用决策模型在强弱 LLM 间路由，成本只降 **7.0–10.5%**，成功随 harness 起伏（有的 −6.0，有的 +7.9）。

**Finding 4：在 agent 状态上训才抬任务成功（Table 3）。** Qwen3-4B 未训 → shifted 任务成功 **13.5%**；+SFT → 步进 52.9→**98.4%**、任务 **96.0%**（孤立准确率只涨约 7 点）；+GRPO 对任务成功几乎不增益（多 4.5 GPU 时 + \$26.53 LLM 费）；再加公开工具数据孤立涨到 85.1，任务反而掉到 **91.5%**。同配方把 StartLux-4B 从 49.0% 抬到 **98.0%**。无 SFT 的纯 GRPO 帮不了未训模型（11.5% vs 13.5%）。

**泼冷水。** 主检验是非劣效（掉分 <3），不是等价；阈值与模型贴着具体 agent/工具/任务族，换家族要重拟合甚至重训；编码 agent 动作是自由文本时决策模型只能选「谁来写这一步」，省得少；硬 DecisionGym（新规则、多客户）上未训规则的 4B 可把成功砸到 **3.0%**（LLM alone 88.0%），重拟合阈值才能救；Jev 失败的 34 个任务都以不可逆错工具开场（常 cancel 该 ask）——自信错选比「偶尔不确定」更致命；English + 多数 Gemini agent LLM，迁移范围需打折。

## 🔬 最有意思的部分

1. **L1→L3 排名翻转是主菜。** 不是「小模型差一点」，而是「孤立题选冠军」会系统性错配 agent 控制权。
2. **REPLACE 与无阈值 CASCADE 是同一类事故。** Figure 5：工具 REPLACE −19.0 / 参数 REPLACE −20.0；拟合 CASCADE 才能「省钱且不掉任务」。
3. **SFT 吃的是分布偏移，不是再刷孤立题。** 步进准确率从 52.9→98.4，孤立只 +7——说明缺口在 agent 轨迹特有状态（含 must-ask、犯错后）。
4. **组合委托要 Stage 1 预注册再 Stage 2 联测。** 文中冻结的是工具+参数 cascade + 反思 trigger；Jev 组合在非劣效上曾差 2.2 点未过，换成 AgentDecision-4B 才以 −81.1% 成本过关。
5. **与 ActWrong 对照。** ActWrong 诊断「事实探针对、动作错」；本文用阈值把不确定交回 LLM，并用 agent 状态重训降低不可逆错工具——两条工程旋钮，不是再堆更大 System One。

## 🤔 我的判断

定位：**部署基础设施 + 评测栈 + 可复现训练配方**。把决策模型从「又一个快裁判」推进到「可共享控制的 harness 组件」。

亮点三条：

1. AgentDecisionBench 三层把「孤立准」和「能控 agent」拆开量；
2. 按决策点拟合阈值的 CASCADE，比全局阈值 / 无阈值 REPLACE 诚实得多；
3. 用 Jev 蒸馏 agent 自访状态，开源 4B 在任务级反超老师。

问题与局限：非劣效不是等价；贴 agent 重拟合成本真实存在；自由文本 coding 步省钱有限；硬规则 OOD 仍脆；良性状态设定未覆盖对抗。

对工程的直接启发：**(1)** 选型决策模型务必跑 L2/L3，不要只看 BFCL/孤立题；**(2)** 生产接入默认 CASCADE + 影子模式拟合 $\tau_d$，禁止无阈值 REPLACE；**(3)** 若要自托管小决策模型，训练集必须含「本 harness 跑出来的状态」，公开工具题不够；**(4)** 编码 agent 别指望决策模型大幅砍成本——更适合客服/家务等多选题循环；**(5)** 和 [ActWrong](./20261015_ActWrong_论文解读_读对却做错_System_One在agent循环里的决策失败.md)、[JustAskJev](../../202609/week4/20260928_JustAskJev_论文解读_对齐失败检测别为每条准则再解码_一次问多题零样本就能筛.md) 叠用：窄问题读事实、不确定升级、阈值门控共享控制，三条一起才闭环。

一句话收束：**能答对孤立选择题，远不等于能安全地替 LLM 开车；JevAgentHarness 的教训是——在 agent 里测、按点拟合阈值、用 agent 自己的状态训，共享控制才从赌运气变成可验收的工程。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
