---
month: 202611
week: 1
date: 2026-11-04
type: 论文解读
slug: CoworkOS
---

# 策略与执行拆开：GLM-5.3 带着 Qwen3.6-35B-A3B 做五轮 RSI，八项均值从 33.19 涨到 42.95

你有没有过这种体验：agent 写完一个演示文稿还行，但要它盯着几天训练、跨会话接着改 CAD，一断上下文就散架？生产端 cowork 其实卡在两件事同时成立——复杂交付要做完，长程监控与决策还要接得上。行业把 Claude Code / Codex / DeepSeek Harness 推得很猛，但多数仍偏「单次把活干完」，对「人与 agent 共同改同一份在制品、中断后还能准确续跑」着墨不够。真正的问题是：**能不能用一套 harness，同时扛复杂生产力任务和持续监控决策，还把人放进环里？**

人大 / 复旦 / 北大等 CoworkOS Team 这篇 **CoworkOS: Scaling Agent Coworkers for Real-world Automation**（arXiv:2611.03178）给出的答案是：把任务策略留给模型，把执行管理留给 runtime；Main Agent + Coordinator + Subagents 三层；共享应用层上人与 agent 改同一份文档 / PPT / CAD / Blender；再加上分层工作区与持久语义记忆。12 项基准上与 Codex、DeepSeek Harness 同台；RSI 案例里以 GLM-5.3 为骨干，对 Qwen3.6-35B-A3B 做五轮自动后训练，八项均值 33.19→42.95（+9.76）。

看完感受：这不是又一个「多 agent 流程图」，而是把 cowork 需要的状态、委托、中断恢复写成一等公民，再用同一套东西去做 RSI 闭环——对 Byron 关心的 harness / RSI 主线，样本价值很高。

## 核心摘要

COWORKOS 是面向长程 cowork 的通用 harness：模型决定动作、委托与验收，runtime 维持状态与反馈。三层架构支持直接执行或同步 / 异步委托给 Subagents，职责边界稳定。共享应用层上，人类既可对话介入，也可直接改正在演化的文档、演示、CAD、Blender 场景。分层工作区与持久语义记忆让工作文件、关键决策、长 Job 日志可跨 agent / 会话共享、复用、中断后续跑。评测在四域 12 项基准（Tool Use / Productivity / Search / ML Coding）上对比 OpenHands、Codex、DeepSeek Harness（DSH），COWORKOS 各域平均都进前二，Productivity 与 ML Coding 最高，整体在四套 harness 里最强。RSI 案例：GLM-5.3 作控制器与数据教师，Qwen3.6-35B-A3B 被训；先用 1,345 条工具轨迹 seed SFT 得到 R0（均值 33.19），再五轮「评估→缺口诊断与数据管线修订→生成校验过滤→SFT→再评估」，R5 均值 42.95（+9.76）；八项里七项超 R0，六项拿到六轮最高（含并列），但 JobBench R5 相对 R0 为 −2.13。

## 论文信息

- **标题**：CoworkOS: Scaling Agent Coworkers for Real-world Automation
- **作者**：Jiajie Jin、Guanting Dong、Chengjun Pan 等（CoworkOS Team；并列一作）
- **机构**：中国人民大学；复旦大学；北京大学；上海交通大学；新加坡国立大学；华东师范大学
- **链接**：https://arxiv.org/abs/2611.03178 （HF Daily Papers 2026-11-04）
- **代码**：https://github.com/RUC-NLPIR/CoworkOS
- **项目页**：https://coworkos-ai.github.io
- **联系**：jinjiajie@ruc.edu.cn；tgui@fudan.edu.cn；dou@ruc.edu.cn

---

## 🎯 为什么这件事值得写

Cowork 叙事常被拆成两半：一半比「能不能把网站 / 幻灯片做出来」，一半比「能不能盯着长 Job」。COWORKOS 明确把两者写成同一设计问题，并补上第三条——人机共享在制品。对 harness 工程这很刺耳：只堆工具调用循环，不够；只堆子 agent，也不够；缺的是**策略与执行的边界、委托生命周期、跨中断的工作区与语义记忆**。

它还把 RSI 后训练当成「终极用例」：失败分析、管线改写、训练决策互相依赖，反馈延迟以轮计——正好同时需要复杂交付与持续监控。用 GLM-5.3 驱动、Qwen3.6-35B-A3B 被提升，等于用 harness 自己跑一遍「harness 能否组织模型改进」。

## 🏗️ 机制：策略在模型，执行在 runtime

### 责任切分

| 角色 | 负责什么 |
| --- | --- |
| 模型（Main） | 动作选择、委托、验收；任务级策略 |
| Coordinator Runtime | Subagent 同步 / 异步生命周期、消息投递、可观察性 |
| Subagents | 有界任务内的局部轨迹与工具使用 |
| 应用层 | 文档 / PPT / CAD / Blender 等共享资源与冲突检测 |

Main 保全局上下文；可独立、上下文很重的子任务丢给隔离上下文的 Subagent，避免把中间步骤全灌回 Main。通信三条路径：Main→Subagent、Subagent→Subagent（经 Coordinator 入 Inbox）、Subagent→Main（同步回调用路径，或异步先 ack 再 Inbox）。

### 共享在制品与工作区

候选产物先在 Subagent 私有区探索，Main 决定是否纳入平台管理的 artifact；基线变了则保留候选再议，而不是默默覆盖。分层工作区 + 语义记忆记录决策、约束、未完成项；长操作以 Job 保留状态与日志，等待或中断后可准确续跑。上下文将满时压缩旧前缀并与 checkpoint 对齐，失败则保留原历史。

### RSI 闭环

每轮：`模型评估 → 能力缺口诊断与数据管线修订 → 数据生成 / 校验 / 过滤 → SFT → 再评估`。训练前用 BM25 任务相似度过滤，控制训评近重复。评估八项：AutomationBench、BFCL v4、OfficeQA Pro、Tau-3 Banking、Harvey LAB、SkillsBench、GDPVal（此处为归一化 rubric，不同于 4.1 的 Elo）、JobBench。

## 🧪 关键证据

### 12 基准四域对比（表 1，GLM-5.3 High thinking）

| 域 | OpenHands | Codex | DSH | COWORKOS |
| --- | ---: | ---: | ---: | ---: |
| Tool Use 平均 | 51.33 | 43.40 | **55.31** | 55.22 |
| Productivity 平均 | 55.27 | 40.11 | 56.13 | **57.54** |
| Search 平均 | **82.22** | 78.78 | 76.64 | 82.14 |
| ML Coding 平均 | 44.66 | 48.17 | 56.14 | **56.26** |

单项上 SkillsBench 64.34、JobBench 59.88、WideSearch 82.47、MLE-Bench-Lite 76.90、MLS-Bench 35.62 等为最高或并列最高。作者结论：四套里整体最强，各域平均均进前二。

### RSI 五轮（表 2，八项 /100）

| Round | R0 | R1 | R2 | R3 | R4 | R5 | Δ |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Automation | 11.00 | 35.50 | 30.83 | 39.49 | 41.83 | 44.15 | +33.15 |
| BFCL v4 | 63.48 | 61.81 | 61.76 | 62.11 | 61.76 | 70.83 | +7.35 |
| OfficeQA | 55.64 | 53.38 | 52.63 | 56.14 | 59.40 | 62.11 | +6.47 |
| τ3-Banking | 14.43 | 19.59 | 16.49 | 18.57 | 22.68 | 22.68 | +8.25 |
| Harvey LAB | 3.33 | 2.50 | 6.78 | 7.53 | 8.42 | 11.08 | +7.75 |
| SkillsBench | 20.85 | 30.38 | 23.18 | 34.16 | 33.84 | 35.76 | +14.91 |
| GDPVal | 74.11 | 75.43 | 77.27 | 75.14 | 76.45 | 76.45 | +2.34 |
| JobBench | 22.66 | 16.93 | 18.27 | 20.58 | 21.32 | 20.53 | **−2.13** |
| **Average** | **33.19** | 36.94 | 35.90 | 39.22 | 40.71 | **42.95** | **+9.76** |

均值从 R3 到 R5 连续上升；增益极不均匀——Automation +33.15，JobBench 反而掉。R0 数据：1,345 seeds；后续轮次混合保留轨迹与新生成（如 R2 含 15.4% 新轨迹等细节见附录）。

### 定性六类应用

前端、游戏、Office、交互可视化、CAD、Blender 与 Codex / DSH 并排检视；作者强调共享编辑与交付完整性，而不只是分数表。

## 🔬 最有意思的部分

1. **策略 / 执行切分写进架构，而不是提示词口号。** Main 拥有验收权，Subagent 拥有有界执行，Coordinator 让交换可持久、可观察——这比「再开一个 worker 线程」更接近生产 cowork。
2. **人机共享在制品。** 对话介入与直接改 CAD / PPT 并列，候选与已发布 artifact 分离，避免探索污染主线。
3. **RSI 当压力测试。** 同一 harness 既跑办公交付又跑五轮后训练；+9.76 有说服力，但 JobBench −2.13 说明「平均涨」≠「全能涨」。
4. **与 DSH / Codex 同骨干对比。** 表 1 不是换模型吹牛，而是固定 GLM-5.3 比 harness——对「模型冻住只改脚手架」这条线（CLEO 等）是互补证据。

## 🤔 我的判断

**定位：长程 cowork harness 的系统样本——策略在模型、执行与连续性在 runtime；并用 RSI 后训练证明同一套机制能组织模型自我改进。**

亮点：

1. 三层委托 + 异步 Inbox 把集体智能与长监控接到同一图上。
2. 分层工作区 / 语义记忆 / Job 日志对准真实中断，不只是 context summarization 话术。
3. 12 基准四域平均全面进前二，RSI 八项均值 +9.76 数字硬。
4. 开源仓库与 showcase，可复现叙事完整。

问题与局限：

- **JobBench −2.13** 提醒多目标后训练会牺牲某些能力；「均值最优」部署前仍要看任务画像。
- **R1→R2 均值 36.94→35.90 回撤**，闭环不是单调的，诊断与数据配方仍是脆弱环节。
- **主对比固定 GLM-5.3**，换弱骨干时 harness 优势会不会缩水，文中未展开。
- **RSI 教师是更强的 GLM-5.3**，被训的是 35B-A3B——增益有多少来自教师质量 vs harness 组织，需要更干净的消融。
- **定性六类应用**有展示力，但评分rubric与交互检查仍偏作者侧；生产迁移要自备验收门。
- **语义记忆与压缩**细节工程重，复制成本不低。

对工程的直接启发：

- 做 cowork 先画清 **谁拥有策略、谁拥有执行状态、谁拥有验收**；别把三者糊进同一个 ReAct 循环。
- 长 Job 用显式 Job 日志 + checkpoint，而不是指望对话历史自己记得住。
- 子 agent 交付「报告 + 引用 + 状态」，Main 再发布；候选与正式 artifact 分库。
- 若用 harness 跑 RSI：每轮必须看分项，而不是只看平均；准备好接受有的基准会掉。
- 与 DeepSeek Harness / Codex 比时，尽量 **固定模型比 runtime**，否则分数不可解释。

一句话收束：**CoworkOS 把「一起把活做完」和「断了还能接着盯」写成同一套 Main–Coordinator–Subagent 与工作区记忆；八项 RSI +9.76 说明这套 runtime 能组织改进，但 JobBench 的回撤也说明它还不会替你做多目标取舍。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
