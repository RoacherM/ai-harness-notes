---
month: 202609
week: 4
date: 2026-09-23
type: 论文解读
slug: Harness-Zero
---

# 把专用 Harness 行为烤进权重：拆掉脚手架，仍能超过挂着它的基线

你有没有过这种体验：Meta-Harness / 技能库 / 中间件把某个域的 agent 抬起来了，一换模型或一拆专用脚手架，增益就蒸发——要么维持「越来越多专用 harness + 路由」，要么接受共享脚手架的次优？北大 / Google / HKUST 这篇 Harness-Zero（arXiv:2609.24974）研究 **agent harness distillation**：训练时用域优化 harness 当指导，把诱导出的行为写进权重，部署时只留一个固定最小目标 harness。关键桥是 **agent-as-harness**——不是把源 harness 轨迹直接模仿（动作空间与信息集不同），而是让「装具智能体」在学生响应边界上审查/最小改正，再在目标 harness 的动作空间里执行。看完我的感受是：它和 [Meta-Harness](../week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 是上下游——那边搜出更好的 \(h^\star\)，这边问 **\(h^\star\) 的增益能不能离开 \(h^\star\) 活下来**。

## 核心摘要

Harness-Zero 三阶段：(1) 在训练任务上演化学生侧 \(h^\star\)（工具/中间件/技能/记忆），并相对固定目标 harness \(h\) **适配**成装具智能体私有参考 \(K=\mathrm{Adapt}(h^\star;h)\)；(2) 学生在 \(h\) 下提案，装具智能体用 \(K\) 做 PASS/REPLACE，改正必须是学生动作空间里的完整响应，审查对话不进学生可见轨迹；(3) 对接受响应做 SFT（mask 审查者视角推理），部署时去掉 \(h^\star\)、\(K\) 与装具智能体。无训练对比：同一演化 harness 下，agent-as-harness 六设定均值 **81.1%**，高于 code-as-harness 的 meta-harness **78.1%** 与裸 mini-SWE **68.6%**。蒸馏到 Qwen3.5-9B：仅 \(h\) 时宏观成功率 **23.3%→44.3%**，超过基座仍挂着 \(h^\star\) 的 **41.7%**；28 个「仅 \(h^\star\) 才有」行为模式平均恢复 **82.3%**。消融刺耳：直接蒸教师轨迹或挂着 \(h^\star\) 的轨迹几乎不涨甚至崩（3–12%），空 \(K\) 审查 11%，给标准答案审查收集成功率 98.6% 却只到 15%，而 \(K\) 指导审查收集 59.4%、测试 **30%**——**收集成功率 ≠ 蒸馏价值**。

## 论文信息

- **标题**：Harness-Zero: Harness Distillation via Agent-as-Harness
- **作者**：Haoran Ye、Yuxing Lu、Haonan Dong、Zhaochen Su、Guojie Song（通讯）
- **机构**：Peking University · Google · HKUST
- **链接**：https://arxiv.org/abs/2609.24974 · 代码 https://github.com/metaevo-ai/harness-zero
- **观察时间**：2026-09-23

---

## 🎯 为什么这件事值得写

Harness 工程（工具、上下文、控制流、持久状态）已是能力杠杆；自动优化（Meta-Harness 等）把脚手架当搜索变量。但增益绑在部署时的那个 harness 上：共享则次优，专用则路由与编排成本滚雪球。联合训权重+harness 的工作又常让增益**继续耦合**脚手架；受控编码研究甚至发现「换评测 harness 比换训练方法更伤迁移」。

Harness-Zero 的野心：把多域专用 harness 发现的行为，沉淀进**同一套权重**，部署只留最小 \(h\)（文中是固定系统提示 + 单 Bash 工具的 mini-SWE-agent）。这和 Byron 做 MMP Pi「Manifest 显式、可版本化」不冲突——脚手架仍要，但**该进权重的程序性行为不该永远外挂**。

## 🏗️ 机制：Agent-as-Harness 当跨动作空间的翻译器

记 \(h\) 为目标、\(h^\star\) 为演化学生侧、\(K\) 为装具私有参考。适配例子：\(h^\star\) 里「拦截危险动作」的中间件，变成 \(K\) 里「学生一提就警告」的审查中间件；工具变成「如何用学生原生动作构造等价物」的规格；技能/记忆变成诊断标准与干预指导。

每步：学生提案 \(y_t\sim\pi_S(\cdot|c_t)\)；装具输出决策 \(d_t\in\{\mathrm{PASS},\mathrm{REPLACE}\}\) 与替换 \(z_t\)；接受响应 \(\tilde y_t\) 经 \(h\) 执行进入下一上下文。约束：装具只读 \(K\)，不能偷看隐藏答案/验证器，也不能直接摸沙箱——额外证据必须通过 \(h\) 可执行动作拿进学生可见轨迹（例如用检查代码替换过早 `finish`）。SFT 目标：

\[
\hat\theta=\arg\min_\theta -\sum_{\tau}\sum_t\log\pi_\theta^h(\tilde y_t\mid c_t)
\]

部署：\(y_t\sim\pi_{\hat\theta}^h\)，系统是 \((\pi_{\hat\theta},h)\)。

三域：SpreadsheetBench Verified（300 训练 / 100 测）、AppWorld（147 训练 / 168 test_normal）、USPTO Retrosynthesis（500 / 100）；Harbor 框架；蒸馏用 GPT-5.6 Sol 当装具、Kimi K3 演化、Qwen3.5-9B LoRA SFT 两轮。

## 🧪 证据：拆掉还能赢，且赢在程序性行为

Table 1（前沿模型、无训练）：带演化 \(K\) 的 agent-as-harness 平均 **81.1%**（相对 \(h\) **+27.6%**）；meta-harness **78.1%**；空 \(K\) 审查仅 **69.2%**——审查本身几乎不解释增益。

Table 2（蒸馏）：

| 设定 | Sheet | AppWorld | USPTO | Avg |
| --- | --- | --- | --- | --- |
| 基座 + \(h\) | 31.0 | 26.8 | 12.0 | **23.3** |
| 基座 + \(h^\star\) | 39.0 | 48.2 | 38.0 | **41.7** |
| Harness-Zero + \(h\) | **44.0** | **58.9** | 30.0 | **44.3** |

DeepAgents / Claude Code 挂在 9B 上宏观还掉点——通用大工具面帮倒忙。作者观察：表格与 AppWorld 的 harness 多是可演示的检查-修改-验证程序，蒸馏后能反超 \(h^\star\)；USPTO 还塞了反应先验与 SMILES 校验工具，蒸馏到 30% 仍落后 \(h^\star\) 的 38%——**深域知识比程序性行为更难烤进小模型**。

Table 3（USPTO 监督源）：教师 rollout / 教师+\(h^\star\) / 学生+\(h^\star\) 蒸完仍约 12% 或崩到 3%；空 \(K\) 11%；oracle 答案审查 15%；Harness-Zero **30%**。挂着 \(h^\star\) 收集的轨迹动作空间不匹配——蒸馏模型会狂调不存在的 harness 工具直到步数耗尽。

## 🔬 最有意思的部分

行为内化（Table 4）：28 个「基座只在 \(h^\star\) 下才出现」的模式，蒸馏后平均恢复 **82.3%**——覆盖 memory / skill / tool / middleware（如先读再改、保护预填格、RDKit 校验、避免重复失败调用等）。这比「总分涨了」更接近「蒸馏到底学到了什么」。

代价诚实：收集时逐步审查让 USPTO 延迟约 **2.4×**（蒸馏后消失）；弱装具模型时 agent-as-harness 可能有害；上下文管理等机制也不总能表达成「学生的一条响应」。未来方向包括世界模型式反事实预测、选择性审查、偏好学习吃 PASS/REPLACE 对。

与同周 [RRSI](./20260923_RRSI_论文解读_Harness_RSI会过拟合正则提案与筛选才保住OOD.md) 对照：RRSI 让**外挂 harness 的搜索别过拟合**；Harness-Zero 让**已经搜到的好行为别永远外挂**——一条管演化动力学，一条管演化产物的落点（权重 vs 脚手架）。

## 🤔 我的判断

定位：**harness 蒸馏的清晰问题陈述 + agent-as-harness 翻译桥**，实验把「动作空间错配」和「答案捷径≠好监督」钉死。亮点三：

1. 部署拆掉专用脚手架仍超「挂着它的基座」，宏观 23.3→44.3 vs 41.7；
2. 无训练时 agent-as-harness 平均压过 code-as-harness，且空 \(K\) 对照干净；
3. 28 模式 82.3% 恢复，比单一 pass@1 更能说明内化。

局限：强依赖装具模型能力；收集贵；深知识与部分 harness 机制仍外挂；9B + 三域是否外推到生产 coding agent 未证。对 Byron（Pi Manifest、skills/MCP、关心 Meta-Harness）：演化环可以继续跑，但关键**重复出现的程序性行为**应进蒸馏候选——部署保持最小 \(h\)，专用 \(h^\star\) 当教师期指导而非终身依赖。建造顺序仍是 Rubric/代码闸门在前；这篇补的是 **「脚手架增益如何回流权重」**，不是替代评测栈。

一句话收束：最好的专用 harness，不该是部署时永远背着的外壳——**Harness-Zero 把它变成训练期的翻译器，让行为在拆掉脚手架之后还在。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
