---
month: 202609
week: 5
date: 2026-09-29
type: 论文解读
slug: EMem-Bench
---

# 长程具身记忆四缺：把历史塞进上下文救不了，空间·事件·场景外存才把 SR 抬二十多点

你有没有过这种体验：coding / 具身 agent 明明「看过」某个文件、某个抽屉、某次失败抓取，下一轮任务却又去开同一把锁、又回床上找已经挪走的书；你把历史轨迹整段塞进上下文，token 暴涨、延迟暴涨，成功率反而掉。浙大 OmniAI 这篇 EmbodiedMemory-Bench / EMem-Bench（arXiv:2609.28236）先手工拆了 Gemini-3-Flash 与 Qwen3-VL-32B 在 EmbodiedBench 上 100 条失败轨迹，发现约 **91%** 可归到四类记忆缺陷，再做了一个 **2,554 集**、四任务族的交互式记忆基准，并给出外存系统 Embodied-Memorizer（空间 / 事件 / 场景）与训过写读环的 EMem-8B。看完我的感受：这和 Byron 盯的 **long-horizon memory / DSSR 写时后悔 / JITMem 读时策展** 同构——只是从文本摘要搬到了「观察–动作–反馈」因果链；和同日归档的 [DSSR](./20260929_DSSR_论文解读_预算够了仍忘_写时后悔才是常驻记忆瓶颈.md) 对读尤其有劲：DSSR 量写时承诺税，EMem-Bench 量「历史证据能不能驱动后续物理动作」。

## 核心摘要

EMem-Bench 要求 agent 在拿到目标任务 \(\mathcal{T}\) 之前，先消化一段多模态交互历史 \(\mathcal{H}\)，再在仿真环境中逐步行动直至终端状态满足 \(\mathcal{T}\)。四任务族分别测：**Passive Observation**（细粒度视觉记忆）、**Dynamic Tracking**（动态世界状态更新）、**Interaction Failure**（交互反馈揭示的不可见状态）、**Experience Generalization**（从多次纠正中归纳规则并迁移）。全量 2,554 集（1,036 / 1,052 / 263 / 203），跨 1,118 场景。主结果：最强全上下文专有模型 Gemini-3-Flash 平均 SR 仅 **64.2%**；开源模型大多 <45%。在 GPT-5.4-mini 骨干上，外存 EMem 把平均 SR 从 35.3% 提到 **58.9%（+23.6）**，并显著低于 MIRIX / MemVerse / TeleMem / MMA；Qwen3-VL-8B 训成 EMem-8B 后相对骨干 **+20.6**。更刺眼：给 GPT-5.4-mini 塞满历史 RGB（token 约 \(4.9\times\)、步延迟 \(9.5\times\)）平均 SR 反而从 36.6% 掉到 **26.2%**——说明瓶颈不是「看得见的历史不够多」，而是「结构化写读」。项目页：https://zju-omniai.github.io/EmbodiedMemoryBench/。

## 论文信息

- **标题**：EmbodiedMemory-Bench: Benchmarking Embodied Memory for Long-Horizon Embodied Tasks
- **作者**：Lizhou Liang、Xinyu Zhong、Miao Pan、Xiaohe Zhou、Xuanyu Liu、Qinfeng Li、Peng Li、Jintao Chen、Xuhong Zhang、Wenqi Zhang 等
- **机构**：Zhejiang University（OmniAI / ACES Lab）· Central South University · Institute of Software, CAS
- **链接**：https://arxiv.org/abs/2609.28236 · https://arxiv.org/pdf/2609.28236 · HF https://huggingface.co/papers/2609.28236 · 项目 https://zju-omniai.github.io/EmbodiedMemoryBench/
- **观察时间**：2026-09-29（HF Daily Papers / AK）

---

## 🎯 为什么这件事值得写

现有记忆评测多落在两类：LLM 多会话检索（LoCoMo / LongMemEval / MemoryAgentBench / Evo-Memory），或具身侧「带着历史做导航 / QA」（FindingDory、SpaMEM、WorldLines、WorldMemArena）。EMem-Bench 的分野是：**记住的东西必须驱动后续可执行动作**，并且把四类能力拆成独立任务族，用等权平均 SR 防止大家族吞掉分数。

手工失败归因（Figure 2）给出工程可对照的四缺：

1. **细粒度视觉遗忘**（约 35%）：物体密场景里看过却找不回；
2. **动态状态未更新**（约 18%）：书已从床→盒子→桌，仍回床找；
3. **交互揭示状态未写入**（约 31%）：锁死的抽屉、失败的抓取——视觉看不见，反馈却告诉了你；
4. **经验泛化失败**（约 7%）：刀和勺都纠正进抽屉了，叉子仍放桌上。

对 Byron（agent memory / harness）：这四条几乎一一映射到 coding agent——「看过文件路径却再搜一遍」「配置已改仍按旧状态操作」「工具报错揭示的约束没进记忆」「同类 API 失败模式不迁移」。同日 [DSSR](./20260929_DSSR_论文解读_预算够了仍忘_写时后悔才是常驻记忆瓶颈.md) 说写时后悔主导；EMem 则用三仓外存强制「写什么 / 查什么」结构化。

## 🏗️ 机制：四任务族 × 历史构造 × 三仓外存写读环

**任务形式。** 每集 \((\mathcal{H},\mathcal{T},\mathcal{A})\)；执行时 \(a_t\sim\pi(\cdot\mid\mathcal{H},\mathcal{T},o^{\mathrm{e}}_{1:t},a_{1:t-1})\)。成功条件是仿真终端状态满足 \(\mathcal{T}\)，不是选择题。另报 **Error Recurrence Rate (ERR)**：是否重演历史本可避免的错误（回旧位置、再开已知锁、违反已学规则）。

**历史构造流水线。** 选多房间场景（AI2-THOR 拼房 / ProcTHOR）→ 构造任务族专属 memory cue → PDDL 插入 distractor 轨迹拉开 cue→task 距离 → 自动可执行性检查 + 人工六维审（证据充分、指令清晰、无泄漏、轨迹合理、cue–probe 对齐、终端判定）。拒绝 143 集后保留 2,554。Experience Generalization 的规律由 Claude-4.6-Sonnet 结构化生成再落地到场景物体。

**Embodied-Memorizer（EMem）。** 三互补仓：

| 记忆 | 存什么 | 典型更新 |
| --- | --- | --- |
| Spatial | 实体图：位置 / 状态 / 关系族（in/on/held_by 互斥替换） | 书从床到桌 → 覆盖旧位置边 |
| Event | 动作–反馈时序 + 多次纠正聚合成规则 | 顶抽屉锁定 → 以后避开 |
| Scene | 带 caption / 实体索引的视觉快照 | 厨房杯 vs 卧室杯 消歧 |

接口六工具：`update_object` / `remember_event` / `remember_scene` 与对应 `query_*`。推理环：**先写历史（无目标任务）→ 任务到达后再查 → 再发 action_sequence**；动作失败或状态大变则打断重开记忆–动作轮。检索：结构化约束 → 词面 → 语义兜底，主集最多 12 条。

**EMem-8B。** 在 ProcTHOR 训练拆分上用教师轨迹 SFT（LoRA，rank 4）：8,852 轮监督（C 写 / M 查 / A 动作），训 Qwen3-VL-8B「写什么、查哪仓、如何依检索行动」。

## 🧪 关键证据

**全上下文仍弱且偏科。** Table 2：Gemini-3-Flash 64.2% 平均 SR；Qwen3-VL-32B Dynamic 仅 20.3% 却 Exp.Gen. 63.1%——偏科说明四能力互补，不是单一「长上下文」。具身特化模型（RynnBrain 3.6%、Cambrian-S 0.8%、RoboBrain2.5 20.0%）并不自动会用长程记忆。

**外存大幅抬升，且压过同类系统。** GPT-5.4-mini + EMem：**58.9%**（+23.6），ERR 从 38.8 降到 24.3；同骨干 TeleMem 37.8、MIRIX 28.2、MemVerse 28.9、MMA 23.4。开源侧：Mistral-Small-3.1-24B + EMem **+16.4**；EMem-8B 相对 Qwen3-VL-8B **+20.6**（其中 Dynamic **+40.3**）。

**更多历史图像帮倒忙。** Table 3 固定 800 集：标准 full_context 36.6% → 塞满历史 RGB 26.2%（−10.4），token \(4.9\times\)、s/step \(9.5\times\)。与「加窗口就能记」的直觉相反。

**三仓消融各有主场。** Table 4：去 spatial 伤 Dynamic 最重；去 event 伤 Interaction / Exp.Gen.；去 scene 伤 Passive。交叉迁移：EMem-8B 在 EB-ALF +5.0，BLINK +1.3，MMMU-Pro / HallusionBench 基本持平。

**失败分析（400 集 GPT-5.4-mini）。** 多数族以 memory-use 错误为主（行为与历史证据矛盾）；Interaction Failure 更多是「知道别开锁却推不出替代方案」的语境推理错——动作执行本身反而少见。

## 🔬 最有意思的部分

1. **Embodied memory ≠ multimodal memory。** Figure 1：文本记忆留语言，多模态留图视频，具身还必须钉住「动作改变世界、反馈揭示约束」的因果链——这正是 coding agent 工具回包该进记忆的理由。
2. **ERR 补 SR。** SR 问终点对不对，ERR 问路径有没有重演可避免错误——和 [TimeEvo](./20260929_TimeEvo_论文解读_专家工具库帮了也害了_失败驱动自进化才敢上工具.md) 的 helped/harmed、[GameArena](../week4/20260928_GameArena_论文解读_主观裁判太吵静态榜又饱和_用胜负当地面真值的对战评测台.md) 的「能程序判就别纯主观」同精神。
3. **写历史时不许偷看未来任务。** 训练与评测都强制 context_ingestion 阶段无 \(\mathcal{T}\)，避免「按答案写记忆」泄漏——搭 memory bench 应抄这条卫生规则。
4. **与 DSSR / JITMem 的分工。** DSSR：写时后悔主导；JITMem：读时策展；EMem：把写读拆成三仓工具环并训策略。工程上可叠——热路径 schema 槽（空间/锁态），冷路径可回放事件，视觉消歧走 scene。
5. **专有模型也只有 64%。** 提醒：别把「换更强 MLLM」当记忆方案；外存 + 写读策略的增益（+16～+24）远大于多数 backbone 微调。

## 🤔 我的判断

定位：**长程具身记忆的测量基础设施 + 可移植的三仓外存基线**（bench 为主，EMem / EMem-8B 为对照系统）。亮点三：

1. 四缺从失败归因写成可执行任务族，等权 SR + ERR 双指标；
2. 「塞更多历史图像反而掉分」直接证伪 raw context 路线；
3. 外存在开源/专有骨干上都涨，且训写读环还有额外 +9.8（相对冻结骨干挂 EMem）。

局限：仿真家庭场景（AI2-THOR / ProcTHOR），动作预算 8 步；Interaction / Exp.Gen. 子集较小；EMem 为 episode-local，跨会话终身记忆未测；项目页在写稿时需确认数据/代码完整开源节奏。对 Byron：

- 给 agent 记忆分层：**空间态（最新覆盖）/ 事件与失败约束（追加+聚合规则）/ 场景消歧**，别只用一条 running summary；
- 合入记忆变更时同时报 SR 类成功与 ERR 类「又犯已知错」；
- 工具失败回包（locked / 404 / permission）必须进 event 仓，禁止只留视觉/文件列表；
- 和 [DSSR](./20260929_DSSR_论文解读_预算够了仍忘_写时后悔才是常驻记忆瓶颈.md) 合读：三仓降低写时自由度，等价于用 schema 压 \(\kappa\)。

一句话收束：**长程具身不是「看得更久」，而是「交互揭示的世界状态有没有被结构化写住、并在下一步动作里用上」**——EMem-Bench 把这句话做成了 2,554 集可跑考场。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
