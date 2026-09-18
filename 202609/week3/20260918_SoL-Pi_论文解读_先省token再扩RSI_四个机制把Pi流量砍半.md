---
month: 202609
week: 3
date: 2026-09-18
type: 论文解读
slug: SoL-Pi
---

# 先省 token 再扩 RSI：NVIDIA SoL-Pi 用可扩展 auto-research 给 Pi 砍掉近半流量

你有没有过这种体验：agent 已经能整夜无人值守跑仓库，但轨迹越长，**重复读大 log、重复回放 tool 结果、edit 后再单开一轮跑测** 把账单吃光——RSI（递归自改进）还没开始，token 先炸了。NVIDIA 这篇 *SoL-Pi: Recursively Scaling Auto-Research Loops for Efficient Agent Harness*（arXiv:2609.20519）把问题钉在 harness 层：在不动权重的前提下，用 **能力约束下的效率搜索** 去挖可迁移机制。研究底盘就是 Byron 栈里的 **Pi**；最终产物 SoL-Pi 是 Pi 的独立扩展（MIT，opt-in）。看完我的感受是：相对本周 Meta-Harness「用完整轨迹搜 harness 代码冲精度」，SoL-Pi 选的是另一条轴——**先把单位任务做便宜，再让下一轮 RSI 搜得更宽**。

## 核心摘要

SoL-Pi 把 harness 改进做成 broad-to-deep 漏斗：外环从约 **152** 个假设方向（context / progress / tools / delegation / prompt-policy / improvement-eval 六族）铺开，内环用独立 disposable lineage + Ralph-style 实现–评审环深挖；能力指标与容差在搜索前冻结且对优化 agent **不可改**，候选必须先过能力门、再至少改进一项效率门，held-out（EdgeBench）只在冻结后评、**永不回灌**。搜索环境约 **535**（495 仓库 issue–PR + 40 verifier-driven），累计 >3000 runs、>60k agent–环境交互。幸存四个机制组成 SoL-Pi：**Action Fusion**（mutation+后续验证同请求）、**Online Context Compact**（在 plan-step 完成边界做经济门控的压缩）、**ObservationPack**（大结果前两次全文、之后 handle+1KB 摘录可分页召回）、**Evidence-Preserving Reducer**（低成本模压 log 成可校验 receipt，失败回退原文）。EdgeBench 51 公开放任务上，相对 Pi：Efficiency 点 **token 流量 −44.7–49.0%、API 成本约 −1/3**，分数保留约 **93.7–94.3%**；Performance 点（按后端选最优单机制）可 **+5.3% / +12.8%** 分并改善 $/score。作者估算相对原生 Codex / Claude Code 约 **\$8.75–\$13.50/小时**，相对 Pi 约 **\$4.36–\$5.71/小时**。20 worker 的 kernel 优化 swarm 里，SoL-Pi 工人相对 Pi 工人 **API −26.8%** 且多过一道速度门槛。

## 论文信息

- **标题**：SoL-Pi: Recursively Scaling Auto-Research Loops for Efficient Agent Harness
- **作者**：Haozhe Liu, Tian Ye, Sensen Gao, Qihang Cao, Yitong Li, Mingchen Zhuge, Duomin Wang, Ruihua Zhang, Ping Luo, Jiawang Bian, Lei Zhu, Ligeng Zhu, Enze Xie, Song Han 等
- **机构**：NVIDIA（NVlabs）
- **链接**：https://arxiv.org/abs/2609.20519 （2026-09-17）· 项目 https://nvlabs.github.io/SoL-Pi/ · 代码 https://github.com/NVlabs/SoL-Pi（Pi 扩展，机制默认关闭）

---

## 🎯 为什么效率要先于「更大的 RSI」

长程 coding agent 的瓶颈已从「会不会写」扩到「每一 token 是否推进工作」。基础设施压每 token 成本、模型侧压参数量——作者走正交方向：**改 harness 如何与环境交互**。难点在于工具、上下文、验证、委派、终止高度耦合，局部省钱常把失败挪到下游；人手翻轨迹又扩不了环境覆盖。更糟的是，近期 held-out 研究指出：进化出的 harness 容易过拟合搜索任务、对未见任务收益很薄——所以 SoL-Pi 三条原则写得很硬：

1. **Breadth + Depth**：先铺假设，再对存活者反复实现–硬化；
2. **Independent validation**：验证失败只拒候选，不触发「对着 held-out 补丁」；
3. **Scalable orchestration**：lineage 隔离可丢弃，失败不串扰。

这和「pretraining the harness」叙事一致：暴露足够多样的可执行环境，从轨迹里沉淀可复用机制，而不是为一个榜特化一坨 prompt。

## 🏗️ 四个幸存机制（对着 agent–环境环不同位点）

### Action Fusion

基线 Pi 常「改文件 → 另开一轮跑测/构建」。Fusion 把 mutation 与 follow-up 合成一次 tool request、一次 observation，去掉中间模型往返；需要先看 mutation 结果再决定的命令仍保持分离。案例研究里该 lineage 记了 **27** 轮：oracle 发现相邻动作模式后，prompt-only 触发不稳，于是**扩展 tool schema 暴露融合动作**，并把 trigger rate 当成中间验收指标——说明 auto-research 会自己发明机制专用度量。

### Online Context Compact

不在全局硬帽上盲压，而在 `update_plan` 的 **subtask 完成边界** 估剩余请求数与窗填充速度，比较「投影输入节省」vs「重写 prompt cache 的额外成本」；未回收的 rewrite 成本会抬高后续门限。过门或逼近窗极限时，才调用 Pi 原生 compaction，并在成功后新开一轮续跑。

### ObservationPack

\>10 KiB 的结果本地归档：前 **两次** provider 请求仍全文；从第三次起换成稳定 handle + 原始大小 + 头尾完整行摘录（约 1 KB），需要时分页精确取回。小结果不动。

### Evidence-Preserving Reducer

对 ≥4 KiB、预定义命令集合的 build/test log：归档原文 → 低成本模抽「证据 receipt」→ **确定性校验**（schema、source hash、exit status、精确引用、体积）；失败/疑似凭据/无缩小则回退原文。Reducer 在 ObservationPack 投影之前跑；Pack 识别 receipt 标记后跳过，避免二次损毁已校验证据。主 agent 仍负责诊断与行动选择。

四者分别打：**动作往返、上下文经济、观察回放、委派阅读**——独立 add-one 都降 token；全栈在两端后端都是最低 total traffic / cost 点。值得注意：缩短上下文会伤 cache-read、抬 cache-write，但 GPT-5.6 Sol 上全栈仍把总成本从 **\$1339 → \$894**——提醒评测要看**整任务账单**，不能只迷信 cache hit。

## 🧪 关键证据：EdgeBench 主战场 + TB4 / IMO / Swarm

**EdgeBench（51 公开题）**（Table 1–2，2026-08-17 价）：

| 配置 | 后端 | Total tokens (B) | Cost ($) | Avg Score | \$/score |
| --- | --- | --- | --- | --- | --- |
| Pi | GPT-5.6 Sol | 2.15 | 1339 | 44.8 | 0.59 |
| SoL-Pi Efficiency | GPT-5.6 Sol | 1.10 | 894 | 42.0 | 0.42 |
| SoL-Pi Performance（ObservationPack） | GPT-5.6 Sol | 2.02 | 1271 | 47.2 | 0.53 |
| Pi | Opus 5 | 2.37 | 1741 | 44.8 | 0.76 |
| SoL-Pi Efficiency（零适配迁移） | Opus 5 | 1.31 | 1158 | 42.2 | 0.54 |
| SoL-Pi Performance（Action Fusion） | Opus 5 | 2.10 | 1605 | 50.5 | 0.62 |

搜索只在 Sol 轨迹上做；迁到 Opus 机制触发率/强度下降，但触发时仍省 token，全栈保留类似 score–efficiency 折中——**跨后端迁移有初步证据，但非对称**。

**Terminal-Bench 4（63 CPU-only）**：Codex/Pi 各解 18，SoL-Pi 解 **15**，但总成本 \$211 vs Pi \$286（−26%），每题成本更低。诚实：效率点会**让出一点绝对解题数**。

**IMO 2026（Lean 4 形式化，6 题）**：SoL-Pi 过 3/6，每题成本最低（\$20.90 vs Codex \$22.89 / Pi \$25.32）。

**Swarm（2h kernel 优化）**：Codex 协调 + 20 SoL-Pi 工人 → **1127 cycles / \$60**；Pi 工人 swarm → 1366 / \$82；单 agent 最便宜但探索更弱。效率 harness 可能把固定预算换成**更密的集体探索**。

## 🔬 最有意思的部分：隔离边界与「递归效率改进」愿景

方法图把「开发反馈」和「held-out」画成硬墙——这是对 harness 过拟合文献的工程回应，也是对 Byron 做 judge / eval harness 的提醒：**验证信号一旦回流搜索，你量到的就不再是迁移**。局限章节提出三条长期方向，都很 Byron：

1. **Pretraining the harness**：继续扩环境与假设多样性；
2. **Multi-backend training**：现在只在单后端轨迹上更新，Opus 触发变稀；
3. **Recursive Efficient Improvement**：用更省的 SoL-Pi 跑下一轮 auto-research，让效率既是产物也是搜索资源——**尚未用本实验证明复利，但是清晰议程**。

开源实现也克制：不 vendor Pi、机制默认关、ObservationPack/Reducer 本地归档、Reducer 远程摘要有安全文档——工程上可渐进接入。

## 🤔 我的判断

定位：**面向 Pi 的可安装效率 harness + 可扩展 RSI 搜索样本**。亮点三：

1. 能力门不可被优化器篡改 + held-out 永不回灌，比「对着榜进化」干净；
2. 四个机制都落在可解释的环位点，且强调证据保全（receipt 校验 / 原文回退）；
3. 直接站在 Pi 扩展 API 上，对 Byron 的 MMP/Pi 栈是「下周就能试」的密度。

问题与代价：EdgeBench 公开子集仅 51；TB4 解题数略降；搜索极贵，文中承认难以在固定预算下系统扫 breadth–depth scaling law；机制在 Opus 上触发更少。和本周 Meta-Harness 对照：Meta-Harness 最大化诊断带宽冲精度，SoL-Pi 在能力容差内最小化流量——**两者应串成外环**：用 SoL-Pi 降低每轮评测成本，用 Meta-Harness 式完整轨迹决定改哪段 harness 代码。

一句话收束：**RSI 的持久价值未必是某一版神器，而是能在公共可执行环境上规模化、挖出可迁移改进的搜索过程——而效率，是让这个过程跑得下去的第一燃料。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
