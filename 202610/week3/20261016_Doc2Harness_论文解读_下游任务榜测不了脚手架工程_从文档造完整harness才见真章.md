---
month: 202610
week: 3
date: 2026-10-16
type: 论文解读
slug: Doc2Harness
---

# 下游任务榜测不了脚手架工程：从文档造完整 harness，才见真章

你有没有过这种体验：SWE-bench / Terminal-Bench 分数涨了，你却说不清是模型变强了，还是 Claude Code / OpenCode / Codex 这层脚手架又被悄悄拧紧了；反过来，你让模型「按文档写一个 coding agent harness」，交出来能装、能跑，一碰权限边界、会话状态、无头驱动就碎——而现有榜根本不直接考「造脚手架」这件事。**下游任务成功，到底能不能代替「会不会做长程 harness 工程」？**

中科院计算所、清华、北大与东南大学这篇 **Doc2Harness**（arXiv:2610.20344）给出否定答案，并交出据称首个**确定性**的长程 harness 工程基准：Stage 1 在固定 builder（OpenCode）里，凭十二家前沿 harness 文档合并改写的开发文档 + 仅接口的合同骨架，造出完整候选；Stage 2 冻结候选、用脚本化 mock 模型端到端驱动，靠四条可观察通道打分，并用 positive control 堵「空过」。看完我的感受：这和仓库里 [Raven](../../202610/week1/20261001_Raven_论文解读_别再手搓更强harness_Harness的Harness才把异构agent当可组合单元.md)、[Meta-Harness](../../202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 关心「改 harness」同一条线，但 Doc2Harness 量的是**从零按规格造出来**——最强 GPT-6 Astra 也只过 **53.05%**，失败还扎堆在「横切功能怎么拆模块」。

## 核心摘要

Doc2Harness 两阶段评测「LLM 会不会做长程 harness 工程」。Stage 1：被测 build model 跑在统一的 OpenCode 1.17.0 builder 里，输入为（1）consolidate 自 Claude Code、Kimi Code、Gemini CLI、Qwen Code、OpenCode、OpenClaw、Codex CLI、Aider、OpenHands、Cline、Crush、Cursor 十二家用户文档的开发文档（条款化、可外部观察、不锁实现），（2）剥自 mini-swe-agent、约七十行 + 三份不可变外部合同（CLI / headless / 评测 API）的合同骨架；在隔离容器内构造，提交时只做合同检查（编译安装、命令齐、无占位 stub、三接口在），通过即冻结，配额最多三提交 / 两小时 / 最多二十四轮续跑。Stage 2：每测例独立容器，脚本 mock 模型驱动完整会话，观察模型请求、磁盘文件、进程退出码、命令输出四通道；130/328 测例至少挂一个 positive control，控制失败则整组记 FAIL，堵 vacuous pass。十九个前沿模型各三独立构建、共 328 测例：GPT-6 Astra **53.05±0.25%** 最高，最弱 Hy3 **27.44±0.43%**，差距约二十六点；均值仅 **38.30%**，56 例全员失败、16 例全员通过。类别上权限 strong-weak gap 最大（**50.8** 点），交互终端近饱和（均 **79.9%**）；模型跨六主类排名可晃 **十六** 位。架构上控制代码行数后，模块数与通过率偏相关 **−0.423**——拆太碎会掉分。价值定位：把 harness 工程从「下游任务代理指标」拉成可复现、可分科的第一性评测。

## 论文信息

- **标题**：Doc2Harness: Benchmarking LLMs on Long-Horizon Harness Engineering of Coding Agents
- **作者**：Yuyao Ge, Shenghua Liu, Peizhong Gao, Chendi Ge, Chenxu Liu, Yiwei Wang, Lingrui Mei, Baolong Bi, Jiayu Yao, Jiafeng Guo, Xueqi Cheng
- **机构**：Institute of Computing Technology, CAS · Tsinghua University · Peking University · Southeast University
- **项目页**：https://geyuyao.com/doc2harness
- **链接**：https://arxiv.org/abs/2610.20344 （2026-10-14）

---

## 🎯 为什么这件事值得写

Harness 决定模型能力有多少落到环境：交互环、工具执行、上下文、权限、会话、失败恢复、结果验收。递归自改进最终也要改这层基础设施。但 SWE 榜的单元测试接口固定、成熟仓库可能进过训练；harness 近年才爆发、改得勤，且「同一功能多种合法实现」「必须被模型驱动才谈正确」——普通 SWE 判据搬不过来。

现有 harness 评测多是：固定任务比不同现成 harness（如 Claw-SWE-Bench 报 harness 选择可挪 **27.4** 点），或固定 harness 测某一能力。另一路让模型改已有 harness 再看下游——增益和「会不会造」缠在一起。Doc2Harness 把 **construction** 和 **black-box behavioral eval** 拆开，并用 mock 模型去掉执行期 LLM 噪声。

| 评测对象 | 典型做法 | Doc2Harness 差在哪 |
| --- | --- | --- |
| 下游 coding 任务 | 固定 harness 比模型 | 不考造脚手架 |
| 现成 harness 对比 | 同一任务换壳 | 不考从规格生成 |
| 改 harness + 下游分 | 反馈驱动修订 | 增益归因不清 |
| **按文档造完整 harness** | Stage1 构建 + Stage2 确定性驱动 | 直接量长程工程 |

## 🏗️ 机制：文档条款 → 合同骨架 → 四通道 + positive control

**开发文档。** 先爬十二家文档抽特征清单，再写成条款（语法 / 含义 / 期望行为），最后消歧：合并同义命令（如 `/compact` vs `/summarize`），拆开同名异义（Claude 的 plan 权限 vs Aider 的 architect/editor 角色）。章节覆盖交付物与运行时接口、内置工具、顶层命令、权限/角色/沙箱、全局 flag、会话内命令、持久记忆、约定文件、跨章一致性、退出码、自动化评测 API。

**合同骨架。** 五份 Python + 安装/入口/配置；代表候选约 30 文件 / 16k 行，约 **99.6%** 由 build model 写出。构建容器无外网（仅指定模型网关），起飞前 preflight。

**测例合成三法。** 公开 issue 挖掘、跨 harness 模块差比对、单变量扩展；设计时锚定条款且求区分力，区分力是否成立事后用数据验。

**Positive control。** 防三种空过：观察前提不成立、约束未真正生效、同形对照会在无约束时也失败。控制本身不计入通过率分母。

**计分与固定项。** Stage 1 统一 OpenCode（避免 Claude/Codex 厂牌自带优势）；Stage 2 统一 mock 脚本。排名按 build model 的测例通过率，不是按某一个候选仓库「好不好看」。

## 🧪 关键证据

**主表（Table 2，摘录）。** 全 328 例均值±三构建标准差：

| 模型 | Perm. | Session | Direct | Interactive | All |
| --- | --- | --- | --- | --- | --- |
| GPT-6 Astra | 71.6 | 44.0 | 39.4 | 91.7 | **53.05±0.25** |
| Opus 5 | 63.4 | 38.1 | 17.2 | 91.7 | 46.65±1.80 |
| Qwen3.8 Max | 61.2 | 43.3 | 15.2 | 93.3 | 45.83±0.14 |
| Grok 4.7 | 54.6 | 28.2 | 22.2 | 78.3 | 43.50±2.49 |
| Hy3 | 28.4 | 19.4 | 13.1 | 65.0 | **27.44±0.43** |
| 均值 | 43.7 | 29.9 | 18.8 | 79.9 | **38.30** |

**边界 vs 普通。** Opus 5 在普通场景 **80.68%** 高于 Astra 的 72.46%，但边界场景 Astra **48.97%**、Opus 5 掉到 **37.02%**——头名差距主要在边界条款。

**类别区分力（Table 3）。** 权限：均 43.7%、strong-weak gap **50.8**、51/61 例有区分力；文件操作 gap 仅 20.9；交互终端均 79.9% 近饱和；direct drive 均仅 18.8% 且 17/33 为 floor。全员失败 56、全员通过 16、真正拉开排名的 256。

**信度。** 57 个候选 × 全部测例 KR-20 = **0.913**。六主类两两相关均正（均值 +0.521），整体通过率与各类相关多在 0.69–0.94——更像一维能力强弱，而非明显「专精某科」；但单模型跨类排名仍可晃十六位（Grok 文件操作第 1、会话状态第 11）。

**架构（Table 4）。** Astra 均 7964 行 / 12.3 模块 / 646 行每模块；Qwen3.8 Max 16681 行 / 40.0 模块；GPT-5.5 仅 1271 行但最大模块占 **78%**。代码行数与通过率相关 0.614（构建级）/ 0.629（模型均值）；**控制行数后模块数偏相关 −0.423**，行/模块 +0.504——横切功能拆太散会伤分。行数方差 95.9% 在模型间，同模型三构建 CV 均仅 12.7%。

**泼冷水。** 最强仍失败近半；56 个 floor 例在本队列无区分力；builder 固定 OpenCode、执行固定 mock，测的是「在这套约束下造壳」不是任意脚手架方言；合同检查过≠行为对，早期提交者通过率是下界；单缺陷可连锁打挂多例；文档来自用户文档合并，不是形式化规格完备性证明。

## 🔬 最有意思的部分

1. **下游榜的「harness 贡献」和「会造 harness」是两件事。** 前者换壳比分，后者从条款生成可驱动系统——Doc2Harness 第一次把后者做成确定性主指标。
2. **空过是一等公民。** 不做 positive control，权限/会话类「看起来过了」会系统性灌水。
3. **模块碎片化有代价。** 不是「文件越多越工程化」；偏相关说的是：同样行数下拆得越碎，越难过横切一致性。
4. **头名会互换。** 普通场景 Opus 5 赢、边界 Astra 赢——只报总分会掩盖「谁更会啃边界条款」。
5. **99.6% 代码由模型写。** 骨架极瘦，逼的是长程实现与模块边界，不是填空补全。

## 🤔 我的判断

定位：**测量基础设施**。给 harness 工程（尤其 RSI / Meta-Harness 路线）一块终于不靠下游代理的考场。

亮点三条：

1. 两阶段冻结 + mock 驱动，把「会造」从「下游碰巧分高」里剥出来；
2. 四通道 + positive control，评测可复现、防空过；
3. 类别切片与架构相关给出可操作的失败叙事（边界条款、模块切分）。

问题与局限：OpenCode/mock 固定带来外部效度问题；floor 例多；文档非形式化完备；未测「改现有 harness」与「从零造」是否同源能力。

对工程的直接启发：**(1)** 宣称模型擅长 agent 基础设施时，别只甩 SWE 分——至少要有行为级 harness 测例；**(2)** 造壳时优先保证横切一致性（权限×会话×驱动），少为每个特性新开零散模块；**(3)** 验收必须带 positive control，否则权限/无头路径极易空过；**(4)** 评 RSI/[Meta-Harness](../../202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 时，把「生成/修订 harness」与「用新 harness 刷下游」分成两段指标；**(5)** 与 [AgentSWE](../week2/20261014_AgentSWE_论文解读_coding_agent交出来看着齐其实空_冻结提交加holdout_rubric才测得出真交付.md) 同逻辑——交得出来、合同检查过，都不等于行为交付可信。

一句话收束：**下游任务榜看见的是「某个壳套在某个模型上好不好用」；Doc2Harness 逼你回答另一句——按规格从零造出那层壳，你到底会不会。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
