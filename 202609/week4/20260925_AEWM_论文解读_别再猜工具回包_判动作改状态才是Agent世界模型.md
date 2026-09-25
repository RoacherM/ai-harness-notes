---
month: 202609
week: 4
date: 2026-09-25
type: 论文解读
slug: AEWM
---

# 别再猜工具回包：判动作、改状态，才是 Agent 世界模型该干的事

你有没有过这种体验：长程 agent 挂着搜索 / 终端 / 写仓工具，轨迹里已经有了真实观测，世界模型却还在学「下一句工具回包长什么样」——网页排名、文件系统噪声、测试输出都是高熵执行依赖，猜对了也未必帮决策；更糟的是，**未经验证的假设、过期计划和「半截进度当完成」会写进历史，污染后续每一步**。人大高瓴这篇 Agent-Editing World Model（AEWM，arXiv:2609.28416）今天冲到 HF Daily Papers #1，核心反转很硬：**别模拟环境观测，去建模「当前推理+动作会把任务状态带向哪」**——用 Action Judge 分 Critical / Exploratory / Noisy，再用 State Revision 在执行前改掉 noisy 的推理–动作续写；EditAct 把改过的状态真正写进轨迹，而不是只当评论家。看完我的感受是：这和 Byron 关心的 **judge-evals / harness 干预点 / coding-agent 长程污染** 同频——世界模型的预测对象从「环境皮」换成了「任务态」，干预从 hint 升级成 **直接改写将进入 history 的那一步**。

## 核心摘要

AEWM 把 pre-execution 状态写成 \(s_t=h_t\oplus(\hat r_t,\hat a_t)\)：历史 \(h_t\) 是已提交证据，当前 \((\hat r_t,\hat a_t)\) 是即将污染未来的解释与意图。传统 \(\mathcal{M}_{\mathrm{obs}}(o_t\mid s_t)\) 在有真工具反馈时价值有限，且仿真可能编造证据。AEWM 学两件事：Action Judge 输出 \(\widehat y_t\in\{\mathtt{critical},\mathtt{exploratory},\mathtt{noisy}\}\)；仅 noisy 时 State Revision 从同一可见历史写出更可推进的 \((\widetilde r_t,\widetilde a_t)\)，再真实执行。训练跨 Search / Terminal / SWE：约 **52B** tokens 的 mid-training + **120K** SFT（AJ/SR 各 60K，三域各 40K）。Action Judge 基准 **3,000** 决策上 macro-F1 **70.5%**，压过最强前沿基线 DeepSeek-V4-Pro **59.9%**（+10.6）。EditAct 在六榜 × 三骨干上相对最强基线平均 **+3.2～+6.7**；Qwen3.5-9B+EditAct（44.1）反超同设置下 35B-A3B 的 ReAct（42.2）。用 EditAct 轨迹做 AEWM-RFT，相对 Self-RFT 再涨 **2.2～2.6** 分、且推理时不必挂在线 AEWM。

## 论文信息

- **标题**：Agent-Editing World Model: Rethinking World Modeling for LLM Agents
- **作者**：Shuang Sun*、Guoxin Chen*、Fanzhe Meng、Jia Deng、Huatong Song、Jinhao Jiang、Wayne Xin Zhao†、Hongteng Xu、Ji-Rong Wen（*共同一作）
- **机构**：中国人民大学高瓴人工智能学院
- **链接**：https://arxiv.org/abs/2609.28416 · Dataset / GitHub（文首）
- **观察时间**：2026-09-25（HF Daily Papers #1）

---

## 🎯 为什么这件事值得写

语言世界模型常继承具身 / 视频线的「预测下一观测」：对物理构型合理，对工具 agent 却尴尬——搜索结果、终端输出、单测日志本就该由真环境给，且难可学。作者点出失败机制 **task-state contamination**：局部看起来合理的一步，把错误解释钉进 history，后续动作在污染态上「合理推进」。观测接地 ≠ 理解更新正确。

对 Byron（coding-agent harness / judge-evals / MCP 工具面）：这把「世界模型」从仿真器叙事拽回 **决策层干预**——和「LLM 只出 critique」不同，EditAct **改写将进入上下文的那对 (r,a)**，再执行；AEWM-RFT 又把干预痕迹烤回权重。适合对照 harness 里的 step gate、critic、Best-of-N。

## 🏗️ 机制：判效果 → 改续写 → 真执行

**Action Judge**：在执行前估决策对后续任务进度的贡献。Critical 填关键缺口 / 拿必要证据；Exploratory 做有信息量的试探；Noisy 会巩固错误假设、过期计划或假完成。输入只有当时可见的 \(h_t\) 与提案 \((\hat r_t,\hat a_t)\)——标签来自成功轨迹的事后标注，但训练目标是 **前瞻** 判断。

**State Revision**：同历史上重写 noisy 的推理与动作，使续写更可能推进；不是另开一条无关采样。

**EditAct**（推理环）：

\[
(r_t,a_t)=\begin{cases}
(\hat r_t,\hat a_t), & \widehat y_t\in\{\mathtt{critical},\mathtt{exploratory}\}\\
(\widetilde r_t,\widetilde a_t), & \widehat y_t=\mathtt{noisy}
\end{cases}
\]

选中动作进真环境，\(h_{t+1}=h_t\oplus(r_t,a_t,o_t)\)。关键：**改的是后续决策所依赖的状态本身**，不是旁路建议。

**训练**：mid-training 混原始轨迹 + 合成 AJ/SR；SFT 用严过滤的 120K 校准两能力。AEWM-RFT：只保留经真实环境验证的高质量 EditAct 轨迹做拒绝采样微调，推理回 ReAct、**不再需要在线 AEWM**。

## 🧪 关键证据

**Action Judge**：总体 macro-F1 **70.5%** vs DeepSeek-V4-Pro **59.9%**；三域 60.9 / 72.1 / 77.8，分别超域内最强约 **+10.3 / +10.5 / +13.4**。Search 最难、SWE 增益最大——说明学的是跨工具形态的决策级判断，不是单环境皮相。

**主表（六榜均分）**

| 骨干 | ReAct | Step Best@3 | Traj Best@3 | **EditAct** | vs 最强基线 |
| --- | --- | --- | --- | --- | --- |
| Qwen3.5-4B | 28.5 | 35.1 | 33.7 | **41.8** | +6.7 |
| Qwen3.5-9B | 34.5 | 38.9 | 38.9 | **44.1** | +5.2 |
| Qwen3.5-35B-A3B | 42.2 | 45.4 | 45.6 | **48.8** | +3.2 |

相对 ReAct 分别 **+13.3 / +9.6 / +6.6**。含 OOD：DeepSearchQA、SWE-Bench Pro。35B 上 BrowseComp 40.9→48.1、TB2 37.1→48.3、Doc2Repo 42.8→48.9。Qwen3.5-Plus 上 BrowseComp 44.1→52.7，Terminal/SWE 增益有限——作者归因 AEWM 与更强 agent 的能力落差。

**AEWM-RFT**（35B，推理无 AEWM）

| | BrowseComp | TB2 | Doc2Repo |
| --- | --- | --- | --- |
| Base | 40.9 / 56.2 轮 | 37.1 / 83.1 | 42.8 / 78.1 |
| Self-RFT | 43.2 / 44.0 | 40.8 / 73.6 | 46.1 / 76.3 |
| **AEWM-RFT** | **45.4 / 39.3** | **43.4 / 69.2** | **48.6 / 89.8** |

前两榜相对 Self-RFT **+2.2 / +2.6**，平均轮次再降 **30% / 16.7%**；Doc2Repo +2.5 分但轮次略增——更像认真验仓而非盲目缩短。

**消融（35B）**：Random Gate / Agent Resampling / AEWM Hint / 只改推理或只改动作 / Self-WM / 用 DeepSeek-V4-Pro 当 WM，均不如完整 EditAct；mid+SFT 优于单阶段（相对 SFT-only：BC/TB2/Doc2Repo **+0.8 / +6.4 / +4.3**）。

## 🔬 最有意思的部分

1. **预测对象换了**：世界模型不问「\(o_t\) 长啥样」，问「这笔决策会不会把任务态带坏」——有真工具时更贴工程。
2. **污染是一等公民**：Search 标注 noisy 约 **60%**，Terminal exploratory **43.2%**，SWE critical **42.1%**——域不同，失败形态不同，在线分布仍保留大格局。
3. **直接改状态 > 评论 / 重采样**：Hint 与 Resampling 都输给 SR；joint 改 (r,a) 优于单组件。
4. **干预可内化**：AEWM-RFT 证明 EditAct 轨迹是可迁移监督，不只是推理时挂件。
5. **小骨干 + 好编辑能压过大骨干裸跑**：9B+EditAct > 35B ReAct——harness 干预点有独立杠杆。

## 🤔 我的判断

定位：**决策层 Agent 世界模型 + 可内化的状态编辑推理范式**，不是又一个观测仿真实例。亮点三：

1. 把 task-state contamination 写成可操作的建模缺口，并对齐「有真反馈时别猜回包」；
2. AJ 超前沿 **+10.6** macro-F1，EditAct 跨六榜稳健，消融把「学门控 + 直接改写」钉死；
3. AEWM-RFT 把在线编辑痕迹蒸馏回 agent，符合「脚手架行为最终可烤进权重」的工程方向（对照 Harness-Zero 等）。

局限：AEWM 骨干与训练数据绑定 Qwen3.5 线，对更强 agent（Plus）增益不均；AJ 标签依赖强模型事后标注与过滤；online 编辑有延迟与误改风险；公开复现依赖 Dataset/GitHub 与附录配方。对 Byron：在 Pi / OpenClaw / coding harness 里，**step gate 不该只做 rubric 打分**——更值得试的是「noisy → 改写将提交的 (think, tool_call)」；评测要显式切 contamination 类型（假完成、过期计划、未验假设），而不是只看终局分；RFT 课表可优先吃 **经编辑且环境验证** 的轨迹，而不是 agent 自玩成功集。

一句话收束：长程 agent 的世界模型，别再跟工具回包较劲——**先判这一步会不会污染任务态，脏了就改写，再让真环境说话**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
