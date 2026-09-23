---
month: 202609
week: 4
date: 2026-09-23
type: 论文解读
slug: ACLArena
---

# Agent 多阶段后训练会忘：共享权重合并有取舍，LoRA 专家路由更近 Oracle

你有没有过这种体验：工业 agent 要依次过 Math → 搜索工具 → 电商决策 → 指令遵循，每个阶段换环境、换奖励、换数据配方——Seq 到电商后，数学从还能考变成近乎裸奔，像把 [ComposeCL](../week3/20260916_ComposeCL_论文解读_持续学习机制组合起来才扛得住百任务记忆.md) 的「机制要组合」从权重世界又骂了一遍？ACLArena（arXiv:2609.23989）给 **Agent Continual Learning（ACL）** 做了可控试验台：在 Qwen3-8B-Base 上跑四阶段流水线，从模型级参数位移与 token 级熵稳定性诊断遗忘，再系统比较 MMOPD / SDFT / 模型合并，最后提出 **MLE = SDFT 骨干 + 每阶段 LoRA RL 专家 + 环境路由**。看完我的感受是：这不是又一篇「加个 replay 就好」，而是把 **SFT 立行为、RL 做小步精修、共享单模合并必有权衡** 写成可操作菜谱——和 Byron 关心的持续学习 / harness 后训练闭环直接相关。

## 核心摘要

ACLArena 先把顺序训练当作显微镜：Math → Search → E-commerce → IF。现象上，前序阶段可有正向迁移，但 **Seq-E-commerce 对先前能力杀伤显著**——文中 AIME 从 Seq-Math 的 **25.83** 掉到电商后的 **6.04**（叙事对照里亦见 Search 后约 23.33→6.04）；NQ 等检索与多跳 OOD 同步塌陷，末段 IF 只能部分捞回。机制上分两层：模型级 PCA 显示各任务 oracle 的参数位移方向彼此岔开，硬揉进同一权重必打架；token 级则是 **低熵 token 极稳、高熵决策点几乎包办改写**（工具调用、查询选择等）。方法比较：多教师混合 on-policy 蒸馏（MMOPD，熵门控 forward KL）、自蒸馏微调（SDFT，离线回放高质量轨迹）、模型合并各有优劣，但 **共享单模巩固存在持续 trade-off**。作者菜谱 **MLE（Mixture of Low-Rank Experts）**：先用 SDFT 快速学会多域行为，再为每阶段训 LoRA RL 专家并用环境路由组合——在 agentic 任务上逼近 per-task oracle，同时更好保住 IF / 数学。

## 论文信息

- **标题**：ACLArena: Agent Continue Learning in Multi-stage Post-training
- **作者**：Haixin Wang、Xiaoxuan Wang、Junkai Zhang、Han Zhang、Renliang Sun、Alexander K Taylor、Yidan Shi、Haoran Deng、Chenguang Wang、Jason Cong、Yizhou Sun、Wei Wang
- **链接**：https://arxiv.org/abs/2609.23989
- **观察时间**：2026-09-23

---

## 🎯 为什么这件事值得写

通用 agent 的工业现实是**多阶段后训练**：CoT、工具、业务环境、IF 很少在同一损失里一次学完。技术报告常给最终菜谱、少给对照；经典 CL 的 replay / 任务模块 / 联合重训又常因环境与奖励管线太贵而用不了。ACL 缺的是：**同一底座、同一课程、把顺序训练、蒸馏、合并放在一张桌子上比**。

相对已归档的 [ComposeCL](../week3/20260916_ComposeCL_论文解读_持续学习机制组合起来才扛得住百任务记忆.md)（机制组合扛百任务记忆）、[CategoryAwareSWE](./20260922_CategoryAwareSWE_论文解读_总分涨着一类却在掉_按类别训专家再蒸馏回来.md)（类别专家再蒸馏），ACLArena 把舞台换成 **agent 多环境后训练**，并明确点出 SFT/RL 分工与「单模巩固的天花板」。

## 🏗️ 机制：诊断 → 三范式 → MLE

**顺序基线**（RQ1）：\(\pi_0\) 起，依次得到 Seq-Math / Seq-Search / Seq-E-commerce / Seq-IF（Seq-Final）。

**诊断**（RQ2）：

- **模型级**：单任务 oracle 在参数位移 PCA 上方向分离 → 异质更新；共享权重里恢复 A 往往伤 B。
- **Token 级**：按 Seq-Math 预测熵分箱——海量低熵位置跨阶段几乎不动；变化集中在高熵少数，对应 agent 轨迹里的决策分叉。

**三范式**（RQ3，各调到作者能达到的最佳形态）：

| 范式 | 复用维度 | 直觉 |
| --- | --- | --- |
| MMOPD | 行为空间 · 在线 | 多阶段教师在学生自生成状态上混合蒸馏；高熵处用熵门控 forward KL，避免把教师不确定的决策点拧成假确定性 |
| SDFT | 行为空间 · 离线 | 各专家轨迹过滤后回放，对学生做自蒸馏式 SFT |
| Model Merging | 参数空间 | 直接合并权重（文中含均匀平均等） |

**MLE**：SDFT 先铺多域行为骨架 → 每阶段独立 LoRA + RL 精修 → 按环境路由专家。发现摘要：**SFT 更新幅度大、方向更一致，负责「立住合法行为」；RL 更新小、偏策略局部，负责「把能力拧尖」**——两者互补，而不是互相替代。

## 🧪 主结果：顺序会忘，单模合并要付账

- **顺序训练**：电商阶段对数学 / 检索的干扰最刺眼（AIME 25.83→6.04 量级）；IF 收尾呈「部分恢复 ≠ 回到 oracle」。
- **巩固范式**：MMOPD / SDFT / 合并都能在某些轴上捞回能力，但表级故事一致——**把异质能力塞回同一共享模型，会留下此消彼长**；没有免费的「全要」。
- **MLE**：agentic 任务接近 per-task oracle，同时 IF / 数学保持更好；相对纯 Seq-Final 与单一巩固路线，更像「骨架共享 + 专家局部」的工程折中。
- **消融线索**：单教师 OPD、SDFT 回放变体与数据配方都会改权衡曲面——说明 ACL 对「回放谁、蒸馏谁」敏感，不能黑盒套默认超参。

## 🔬 最有意思的部分

1. **遗忘不是均匀抹除**：低熵续写几乎不动，高熵决策点被后续阶段重写——这把「灾难遗忘」从层范数叙事拉回 **agent 动作选择点**，对工具调用密集的 harness 尤其值钱。
2. **MMOPD 的熵门控**：在高熵决策点，教师本身不确定，硬 KL 模仿等于伪造自信；门控是承认「agent 轨迹里有一簇等价可行策略」。
3. **SFT vs RL 的角色**：和「全程 RL 万能」流行叙事相反——多域 ACL 里更像 **SFT 定行为流形，RL 在流形上微调**；LoRA 专家则避免大步更新互相踩踏。
4. **与 CategoryAwareSWE 同构**：那边是 SWE 类别专家再蒸馏；这边是阶段专家 + 路由。差别在环境/奖励异构程度——agent ACL 更脏，也更接近真实部署。

## 🤔 我的判断

定位：**Agent 多阶段后训练的诊断型试验台 + 可落地的专家路由菜谱**。亮点三：

1. 把现象（会忘 / 会迁）钉到模型位移与 token 熵两层机制；
2. 公平比较 MMOPD / SDFT / 合并，并承认共享单模的 trade-off；
3. MLE 把「SDFT 骨架 + LoRA RL 专家」写成可复用默认，而不是又一个 acronym。

局限：课程顺序固定（Math→Search→E-comm→IF），换序可能改故事；底座 Qwen3-8B-Base，外推到更大稀疏模型待验；路由依赖环境信号，开放域如何自动选专家未闭合。对 Byron（持续学习 / harness / 后训练闭环）：若你在叠能力，**不要默认「最后一个 checkpoint 通吃」**——至少保留阶段专家或可回放轨迹；评估要看高熵决策点，而不是只看平均 token 准确率。和 ComposeCL 拼：那边谈记忆机制组合，这边谈 **后训练阶段组合与单模巩固的物理上限**。

一句话收束：Agent 持续学习的一阶事实是 **决策点会被下一阶段改写**；共享权重想全要，就得接受权衡——**骨架用 SDFT，尖刀用路由 LoRA**，比幻想一个万能 Seq-Final 诚实。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
