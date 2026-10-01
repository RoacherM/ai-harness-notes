---
month: 202610
week: 1
date: 2026-10-01
type: 论文解读
slug: Raven
---

# 别再手搓更强 harness：「Harness of Harnesses」才把异构 agent 当成可组合智能单元

你有没有过这种体验：Claude Code 修仓很猛、Codex 写脚本顺、OpenClaw 走 ACP 也能挂上——可一旦目标跨研究 / 编码 / 设计 / 运维，你要么把所有能力塞进**一个**超重 harness，要么靠聊天室式多代理「自由发挥」，结果是交接丢文件、依赖画不清、失败不知道该重派谁。EverMind AI 这篇 **Raven**（arXiv:2609.33439，仓库 EverMind-AI/Raven）把中心问题从「怎么给单一域焊更强脚手架」拧成：**如何自动构造、用经验进化、再跨域编排多种可执行 model–harness 对**。看完我的感受：这正好踩在 Byron 的 **multi-harness bridge · OpenClaw/ACP · control-plane · skills distill** 交叉点上——不是又一个「角色扮演 MAS」，而是把第三方 harness 当一等公民，用 Host + 显式 DAG + Skill Forge / EverOS 做成「脚手架的脚手架」。

## 核心摘要

Raven 自称 **The Harness of Harnesses**：每个可执行的「模型 + harness」是可组合智能单元；内置 Raven-Research / Code / Design / Oncall，并通过执行适配器挂 Claude Code、Codex、Hermes、OpenClaw 等（含 **ACP** / CLI / 进程内 loop）。**Host Agent** 把目标拆成带依赖的 DAG，运行时先做准入校验（格式 / 图结构 / 能力 / 状态 / 环境），再按就绪集调度、独立 judge 判定节点是否 accomplished、异常回传 Host 续跑 / 放弃 / 重规划；制品以文件账本交接，避免自由文本复述丢细节。经验侧：**host archive + EverOS** 跨任务记忆，**Skill Forge**（承接 SkillCorpus）把轨迹经验变成可检索程序；**HarnessBank 式自进化**在冻结任务模型下诊断失败、改 prompt/knowledge/runtime/config 四类策略面。理论章给出「合同实现 + 兼容计划 + 预算」下的可靠组合充分条件，以及互补局部能力时覆盖可超出任一单体 agent 的推论。实证：自建 **MAOB**（140 题、四域组合全覆盖）上 Raven 在两骨干上四项图指标全第一，Exact Match 相对最强基线 **+10.4 / +10.5 pp**；HarnessBank 七榜 frozen Qwen3.6-27B 全涨（AppWorld +15.4 等）；Research / Code / Design / Oncall 与 SkillCorpus 挂载亦有对照增益。

## 论文信息

- **标题**：Raven: The Harness of Harnesses for Composable Agentic Intelligence
- **作者 / 机构**：EverMind AI（https://evermind.ai/）
- **仓库**：https://github.com/EverMind-AI/Raven
- **链接**：https://arxiv.org/abs/2609.33439 · https://arxiv.org/pdf/2609.33439
- **观察时间**：2026-10-01（HF Daily Papers 前列；multi-harness / Skill Forge / EverOS）

---

## 🎯 为什么这件事值得写

Agent 叙事已从「单模型加工具」走到「专用 harness 吃透一域」，下一堵墙是**跨域交付**：FPS 游戏要从调研到玩法、视觉、集成、运维串起来；每段工具、完成判据、制品形态都不同。手搓一个全能 harness 不可扩展；把各域 agent 丢进同一对话循环，又缺**可验证的交接合同**与**能力注册表**。Raven 的分野在三点：（1）异构 harness 经适配器进同一编排面；（2）计划是**先校验再调度**的 typed DAG，不是事后复盘聊天记录；（3）编排之外还有 **harness 进化 + 技能锻造 + 持久记忆**，把「一次跑通」变成可复用资产。对关心 OpenClaw / ACP / Pi·MMP 的人：这是「多脚手架桥接」的系统级样本，而不是又一份 prompt 角色卡。

## 🏗️ 机制：编排、进化、记忆与技能

**可组合单元。** \(a_i=\mathrm{Agent}(M_i,h_i)\)：harness 含工具、上下文/记忆、技能、控制规则与执行接口。能力定义为共享预算 \(B\) 下可靠任务覆盖 \(p_S(t;B)\)。Host 提交计划 \(\pi=(G,\sigma,\Phi,\Gamma,\lambda,\mathbf{b})\)——DAG、执行者赋值、输入装配、合同/不变量、调度序与预算向量。兼容计划要求前置条件可启用、转移保持不变量、终态满足任务验证 \(\mathsf{V}_t\)。

**Host 五阶段协作。** 按需加载编排指南 → 准入校验（Table 1：路径沙箱、无环、stateful 实例、禁止子代理内递归委托等）→ 依赖就绪调度（未裁决完成的前驱会挡住消费者）→ 完成裁决（独立模型读 prompt/输出/转录尾；失败分类如缺凭据、工具挂、输出截断）→ 运行报告回 Host 做终合成。澄清请求先由 Host 用会话与 EverOS 自动填，**从不代用户批准推送/删除/支付**。

**记忆与组记忆。** 制品路径驱动依赖释放；记忆蒸馏异步。可选 shared group memory：Host 写 round summary / per-agent verdict（close + 下一用户轮 feedback），dispatch 时按相关性 + 近因 prefetch——默认关闭、文中未做效果评测（作者诚实标注）。

**Harness 自进化（HarnessBank）。** 四策略接口 Mem / Plan / Cap / Act 可编辑，评价与簿记内核冻结；Evolver 从失败轨迹出诊断 → 补丁 + 激活规格 → 有效性/激活/配对增益筛选 → gene bank 按「编辑类 × 病理」存精英。任务模型始终冻结。

**Skill Forge。** 本地技能 + 记忆衍生技能 + SkillHub（SkillCorpus 策展目录）；按任务检索可执行程序。第三方 Claude Code / OpenClaw 等同栈可挂同一目录做对照。

## 🧪 关键证据

**MAOB（规划隔离、不派工人）。** 140 职业向请求 × 四域（research / coding / content / oncall）参考 DAG；指标 Node F1 / Edge F1 / POA / Exact Match。Qwen3.8-27B：Raven Exact Match **0.711** vs 最强基线 0.607（+10.4 pp），Node F1 0.923 vs 0.776；DeepSeek-V4-Flash：Exact Match **0.867** vs 0.762（+10.5 pp）。基线为同骨干下的 Claude Code 与 Hermes Agent。Edge 全面低于 Node——**依赖预测仍是难点**，但 Raven 在边上仍领先。

**HarnessBank（冻结骨干）。** Qwen3.6-27B held-out Pass@1：AppWorld 41.3→56.7、BrowseComp+ 16.9→30.8、LiveCodeBench 58.1→71.8 等七榜全升；SWE-bench Verified 小样本未过配对显著性。

**专家与技能。** Raven-Research 在 DeepResearch Mixed 上三骨干相对最强基线约 +6.6 / +3.3 / +7.6 pp；Code / Design / Oncall 在各自榜上有小到中幅增益（文首总览图）。SkillCorpus 挂载相对 no-skill：SkillsBench / GDPval / QwenClawBench 上 Raven 与 OpenClaw 均受益，Raven 在其中两榜增益更大。

**泼冷水。** 理论是充分条件，平均成功率≠合同可靠性前提已满足；MAOB 只测规划图一致、不证端到端正确；组记忆默认关且未评；参考图由作者流水线构造，存在「图先于题」的标注偏差风险；部分商业对照有历史观测偏差说明。

## 🔬 最有意思的部分

1. **第三方 harness 是一等公民，不是二流子代理。** Claude Code / OpenClaw 经 ACP 进注册表，能力标签（stateful / files / progress）直接约束规划——这比「再写一个统一 loop」更贴近真实工具链。
2. **准入校验在派工前失败，避免半吊子执行。** 拒绝时带回编排指南指针——把「schema 合规」做成 Host 的可学习反馈，而不是静默崩。
3. **完成裁决 ≠ 客观正确。** Judge 超时 fallback 当 accomplished——作者明确说域测与终制品检查才是独立证据；理论里的 \(\mathsf{V}_t\) 与运行时 judge 刻意拆开。
4. **互补比特玩具例子。** 两 agent 各只读一位、Host 合成异或——在预算内 \(p_{S_H}=1\) 而单体 ≤1/2，把「组合为何能扩覆盖」钉成可检查构造，而不只是口号。
5. **Skill Forge 与 harness 进化分属「程序复用」与「策略面搜索」。** 一个改可调用技能目录，一个改围绕冻结模型的执行政策——对「skills distill vs RSI harness」两条线都有落点。

## 🤔 我的判断

定位：**开放异构多 harness 编排生态系统 + 组合充分条件理论 + MAOB 规划基准**（系统/基础设施论文，附专家与技能实证）。亮点三：

1. 把「可组合单元」落成注册表 + 适配器 + typed DAG，而不是聊天角色；
2. MAOB 把编排质量从端到端分数里拆出来，Exact Match +10 pp 级差距可读；
3. ACP / OpenClaw / Claude Code 同框，直接服务多脚手架桥接叙事。

局限：端到端「新任务覆盖」需更强单体对照与成本会计；组记忆与部分 Oncall 数字偏内部；超长理论附录工程可读性一般。对 Byron：**(1)** MMP / Pi 若要接多 harness，优先抄「能力标签 + 派工前校验 + 制品路径交接」，而不是再堆角色 prompt；**(2)** OpenClaw/ACP 路径已在 Raven 预设里——桥接层应暴露 stateful/files/progress，而不是只透传字符串；**(3)** Skill Forge 方向对齐 skills distill：失败轨迹 → 可检索程序，但要配**完成合同**（judge/evals）防「看起来像技能」；**(4)** HarnessBank 式进化适合当 control-plane 外环：冻结骨干、改策略面、gene bank 按病理分格。接上周 [Meta-Harness](../202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) / [Harness-Design](../202609/week3/20260918_Harness-Design_论文解读_脚手架不是一整块_组件要按模型和预算配.md)：Raven 补的是**跨异构执行器的编排与复用操作系统**。

一句话收束：**更强的单体 harness 越难手搓——Raven 用「Harness of Harnesses」把 Claude Code / OpenClaw 等变成可调度、可进化、可技能化的组合单元。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
