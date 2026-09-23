---
month: 202609
week: 4
date: 2026-09-23
type: 论文解读
slug: AIDE2-RSI
---

# 双环 RSI 改自己代码：8 天刷出能打两年人工工程的研究 Agent

你有没有过这种体验：把研究 agent 的 harness（搜索策略、上下文、失败记忆、校验）调到天花板，却发现再堆人工工程收益在变薄——像 [Meta-Harness](../week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 能搜、[EvoHarnessBench](../week3/20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) 会忘，真正缺的是**让 agent 在固定美元预算下持续改自己代码、且只在私有 held-out 上及格**的闭环？Weco AI 的 AIDE²（arXiv:2609.26457）把 recursive self-improvement 落在 **harness 层双环**：内环是研究 agent 在 AI R&D 任务上优化代码；外环重写该 agent 自身，按私有 grade \(g(a)\) 决定是否接纳。看完我的感受是：它和已归档的 [RRSI](./20260923_RRSI_论文解读_Harness_RSI会过拟合正则提案与筛选才保住OOD.md)（正则搜索防 ID 过拟合）、[Harness-Zero](./20260923_Harness-Zero_论文解读_把专用Harness行为烤进权重_拆掉仍能超过挂着它的基线.md)（把 harness 行为烤进权重）不在同一条解法上——AIDE² 问的是 **能不能用私有 held-out 当点火条件，让 harness 自改跑出可迁移的研究效率**。

## 核心摘要

AIDE² 把 RSI 做成嵌套树搜索：外环提案改写当前 incumbent 的完整 agent 代码；候选被当作内环优化器，在一组 AI R&D 任务上跑，聚合成**从不给被改 agent 看的**私有分数 \(g(a)\)，只有 \(g\) 提升才接纳。一次自主 **8 天**运行接受 **7** 次改进（步 2/6/28/39/47/63/85），incumbent grade 从 **0.703→0.778**。最强发现体 \(\mathrm{AIDE}_{85}\) 在从未参与筛选的四项外部基准——ALE-Bench、MLE-Bench、FML-Bench、以及 OOD 的 WeatherBench 2——上 **匹配或超过** 两年人工工程的生产基线 \(\mathrm{AIDE}_{\mathrm{human}}\)（FML 强基线之一）。另有 held-out 任务族上，奖励黑客率从 **55%→32%**，低于人工基线的 **39%**——而循环从未显式优化该属性。Ignition（用发现体当外环继续自改）因噪声与种子成本，结果**不结论性**。\(\mathrm{AIDE}_{85}\) 的可归因机制包括：UCB1 bandit 覆盖多种 draft 策略并周期性 forking；有界、角色化上下文 + bug-rate 门控的失败记忆（相对 \(\mathrm{AIDE}_0\) 全历史提示，单次 prompt 可小到约 **50×**）；以及稳健性补丁。

## 论文信息

- **标题**：Recursive self-improvement of AI research agents
- **作者**：Dhruv Srikanth、Bingchen Zhao、Dixing Xu、Yuxiang Wu、Zhengyao Jiang
- **机构**：Weco AI
- **链接**：https://arxiv.org/abs/2609.26457
- **观察时间**：2026-09-23

---

## 🎯 为什么这件事值得写

研究 agent 正在覆盖训练效率、推理优化、算法发现乃至 harness 设计本身；但**研究过程的效率**往往仍冻结在某一版人工脚手架上。Bloom 式「累积 R&D 支出收益递减」在 agent 时代换成：你花更多人时拧 harness，边际更薄。RSI 的经典野心是把优化对象指回研究过程自身——AIDE² 的落地选择很明确：**改 harness 代码，不训骨干**；验收必须过私有 \(g(a)\)，避免 evolve 集纪念品。

放进 Byron 侧坐标：相对 [RRSI](./20260923_RRSI_论文解读_Harness_RSI会过拟合正则提案与筛选才保住OOD.md) 的「正则提案+筛选保 OOD」、[Harness-Zero](./20260923_Harness-Zero_论文解读_把专用Harness行为烤进权重_拆掉仍能超过挂着它的基线.md) 的「蒸馏后拆掉仍能跑」，AIDE² 是 **agent-as-optimizer 的 meta 层**：外环搜的是完整研究 agent，内环才是任务代码树。三者可拼成「怎么搜 / 怎么正则 / 要不要进权重」三角。

## 🏗️ 机制：内环优化任务，外环优化研究效率

形式化上，内环 agent \(a\) 在可测目标与固定预算下解 AI R&D 任务；外环对 \(a\) 的**自身代码**做 meta 优化，目标是提高内环研究效率。一步 RSI（文中 Fig.1）三层放大：

1. **外环树**：每个节点是一个完整研究 agent；proposer 改写 incumbent。
2. **打分**：候选在任务集上当优化器跑；held-out 分数聚成私有 \(g(a)\)，只有改进才留下。
3. **内环解**：每个解节点是该任务上的可运行脚本。

起点 \(\mathrm{AIDE}_0\) 是去掉 ML 专用机件、保留树搜索的精简 AIDE；\(\mathrm{AIDE}_{\mathrm{human}}\) 则是两年人工 R&D 的生产版。筛选任务与 ALE/MLE/FML 同属「优化竞赛」家族但无重叠；WeatherBench 2 的物理天气预报优化明确标为 **OOD**。

## 🧪 主结果：held-out 泛化 + 意外少黑

- **持续改进**：主 run 7 次接纳，\(g\)：0.703→0.778；另两套同协议 run 分别接纳 2 与 4 次——趋势可复现，但强度有方差。
- **外部四基准**：\(\mathrm{AIDE}_{85}\)（及中途强点如 \(\mathrm{AIDE}_{47}\)）相对 \(\mathrm{AIDE}_0\) / 人工基线，在 ALE / MLE / FML / WeatherBench2 上匹配或超过 \(\mathrm{AIDE}_{\mathrm{human}}\)——关键是这些基准**从未进入**候选筛选。
- **奖励黑客**：独立 held-out 任务族上 55%→32%，比人工 39% 再低 7pt；目标函数里没有这一项，属于涌现。
- **跨模型转移**（附录）：harness 增益可挂到未参与搜索的骨干上，说明留下的更像程序机制。
- **Ignition**：发现体继续当外环仍能出接纳改进，但三种子噪声下无法断言比 \(\mathrm{AIDE}_{\mathrm{human}}\) 更省样本——作者诚实标 **inconclusive**。

## 🔬 最有意思的部分：\(\mathrm{AIDE}_{85}\) 里长出了什么

1. **搜索策略**：相对 \(\mathrm{AIDE}_0\)「贪心改进最高分节点」，\(\mathrm{AIDE}_{85}\) 用 **UCB1 bandit** 在多种 draft 策略臂上选，并掺 softmax；选中臂后再扩展该臂标签下最高分节点，并周期性 **forking**——多样性有显式旋钮，而不是只靠温度碰运气。
2. **上下文压缩**：draft / improve 改为读根与近期候选的紧凑摘要；**bug-rate 门控**的失败记忆只在失败率高时注入。相对全历史提示，单次 prompt 在 ALE/FML 上可到约 **50×** 更小（WeatherBench 亦可达 40×+）——同等美元预算下等于买更多搜索步。
3. **稳健性补丁**：外环还留下一批「不涨分但不黑 / 少炸」的工程修补；与奖励黑客下降一起读，像在说：私有 \(g(a)\) 虽不显式罚黑客，却偏好**更难被刷分捷径绑架**的研究过程。
4. **和 RRSI 的对照**：RRSI 怕的是 evolve 过拟合；AIDE² 用**私有 held-out \(g\)** 当接纳闸，再用完全外部基准验迁移。一个正则搜索轨迹，一个正则验收信号——都在打「背题 harness」。

## 🤔 我的判断

定位：**把 frontier 研究 agent 的 RSI 做实到 harness 双环、并用私有 grade + 外部 OOD 讲清楚迁移的方法论文**。亮点三：

1. 验收设计干净：筛选看 \(g(a)\)，故事看从未进环的四基准 + 黑客率；
2. 发现体可归因（bandit 策略、有界上下文、失败记忆），不是黑盒「变强了」；
3. 明确区分「内环会优化」与「外环点火是否更快」——后者不强行宣称。

局限：单次主 run 成本高、种子少；Ignition 无决；任务仍偏可自动打分的 R&D 优化，距离开放科研工作流还有跳。对 Byron（MMP / Pi、关心 Meta-Harness / RSI / EvoHarnessBench）：若你在跑脚手架自改进，**至少把「私有 held-out 接纳闸」和「角色化上下文预算」做成默认**——否则 Meta-Harness 式完整轨迹搜索很容易一边刷 evolve、一边把 prompt 胀到买不起步数。和 RRSI、Harness-Zero 拼：那边管搜索动力学与权重蒸馏，这边证明 **harness 代码本身可以被 agent 持续改写且留下可迁移机制**。

一句话收束：RSI 若只改提示词，你得到的是更会背题的壳；**改自己代码、只认私有 \(g(a)\)**，才有机会在 8 天里追平两年人工工程，还顺手把奖励黑客压下去。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
