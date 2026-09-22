---
month: 202609
week: 4
date: 2026-09-22
type: 论文解读
slug: CategoryAwareSWE
---

# 总分涨着、一类却在掉：按类别训专家，再蒸馏回一个 SWE 策略

你有没有过这种体验：联合 RL 把 SWE 总分抬了一截，拆开一看却是「A 类涨、B 类掉」——aggregate resolution 把跷跷板盖住了。阿里这篇 *One to More, More to One*（arXiv:2609.23377；亦见今日 HF Daily Papers）把这种现象叫作 **category see-saw**，并给出一条 **category-aware** 路线：先按类别把「一个」拆成「多个」专家（One→More），再用 label-routed MOPD 合成「一个」可部署学生（More→One）。看完我的感受是：它不是再堆一条 pooled recipe，而是把**任务分布的结构**写进训练与评测协议——对 coding-agent 后训练，这是从「只看总分」迈向「看最弱类」的关键一跳。

## 核心摘要

框架两段。**(1) Refresh–Repair–Expand（RRE）**：SWE Labeler 做证据接地的多轴标注，把仓级任务组织成可审计类别；同起点的类别专家交替跑长程 Agentic-miniRL 与 RRE——用更新后的策略刷新实例掌握度、把本专家验证过的成功轨迹做 Repair SFT、再扩选下一轮 RL 任务。**(2) MOPD**：标签路由的多教师 on-policy distillation，配 ReLU-gated reward extrapolation，只保留相对参考策略「变好」的教师方向；专家训练与整合都**不依赖外部模型**提供解轨迹或动作目标。评测对照 Pooled RL / Balanced RL，看总分、分项、相对联合 RL 的 **minimum category lift** 与 expert-gain recovery。最终学生 **Logics-SWE-Qwen3.6-27B**：Pro-618 均值分辨率 **52.64%→58.04%（+5.40）**，SWE-bench Multilingual **56.22%→59.00%（+2.78）**；同表上 Pooled / Balanced 大约停在 55% 量级，MOPD 明显更高一档。

## 论文信息

- **标题**：One to More, More to One: Category-Aware Iterative Expert Training for Software Engineering Agents
- **作者**：Jie Zhao*†、Ziyu Jiang*†、Suhang Zheng、Minghui Shan、Xiaoxiao Xu、Lin Qu（*共同一作；†通讯）
- **机构**：Alibaba Group
- **链接**：https://arxiv.org/abs/2609.23377 （v1，2026-09-20）· 模型 https://huggingface.co/Logics-MLLM/Logics-SWE-Qwen3.6-27B · 数据子集 Logics-SWE-Env-2.5K（2,553 条公开）
- **观察时间**：2026-09-22（HF Daily Papers）

---

## 🎯 为何「总分」会骗你：category see-saw

仓级 SWE 表面上共享「可执行裁决」接口，实际需求差很大：服务/数据层修 bug、面向用户的界面改动、基础设施与工具链、性能或安全补丁，各自吃不同证据、工具交互与验证路径。现有后训练大致三类缺口：

1. **联合混合训练**（SWE-RL / SWE-Gym / SWE-Master 等）：方便，但压制对优化真正重要的结构——局部逻辑修复与跨模块兼容、构建修复、安全补丁在轨迹长度、奖励方差、数据频率上本就不一样；一次联合更新可能帮一类、伤一类，而单一 aggregate 看不出来。
2. **流水线子任务分解**（Agentless、SWE-Fixer、AutoCodeRover）：按定位/修复等功能阶段切。本文的切法互补——切的是**训练任务分布的类别**，不是 issue 工作流阶段。
3. **跨域专家再融合**（BTM、MOPD、ExOPD）：通常域边界现成（数学/代码/指令）。本文问：在**同一个**仓级 SWE 域内，用可观察标签建立类别粒度，再走「专精→整合」。

见图式动机：Pooled RL 在全量 **6,723** 任务混合上训一个策略、固定 Pro-618 上拆类看，会出现类别间此起彼伏——这就是 see-saw。Balanced RL 按类分层采样（文中每类 516、合计 1,548）缓解暴露不均，但仍是单策略；真正的「更强专家」需要显式巩固成功行为与策略自适应选任务——于是有 RRE。

## 🏗️ 机制：Labeler → RRE 专家 → MOPD 学生

**可执行任务池与 SWE Labeler。** 构造约 **32K** 可执行候选；主实验从 Domain L1 规则冻结出三类操作分组（与评测对齐）：

| 类别 | 含义 | Pro-618 规模 |
| --- | --- | --- |
| Pro-A | service / data / security | 221 |
| Pro-B | user-facing applications | 201 |
| Pro-C | systems / tooling / runtimes | 196 |

Pro-618 是对 SWE-bench Pro **731** 题的公开审计子集（618）；Multilingual 用全量 **300** 题（A/B/C 为 72/25/176，另有 27 未路由实例仍计入 Full）。训练侧 6,723 里映射进 A/B/C 的有 2,769（601/516/1,652），其余在池外或未映射。

**RRE（One→More）。** 所有专家同起点；Agentic-miniRL 提供共享长程 RL 配方。Refresh 用新 rollout 刷新掌握度；Repair 只复用**本专家上一轮 RL、经 verifier 批准**的成功轨迹做 SFT；Expand 在更大候选池上找新有信息量的任务（含历史上 recoverable-zero），把「会了什么 / 还该练什么」写进课表，而不是死磕固定集合。

**MOPD（More→One）。** 学生从公共基座出发、**自己生成**轨迹；每条训练任务按标签路由到对应类别教师做 token 级监督；参考锚定的外推用正的 teacher–reference gap 门控。整合阶段是纯蒸馏——**不再**加环境奖励项。部署时只要一个模型，不需要类别标签或分专家推理。公开权重即 Logics-SWE-Qwen3.6-27B（Apache-2.0）。

## 🧪 主结果：专家路线压过两种联合 RL

HF 模型页与论文一致的三轮均值（每题每轮一个候选补丁；± 为 population SD）：

| 模型 | Pro-618 | SWE-bench Multilingual |
| --- | ---: | ---: |
| Base（Qwen3.6-27B 研究基座） | 52.64 ± 0.28 | 56.22 ± 0.68 |
| Pooled RL | 55.50 ± 0.46 | 55.56 ± 0.83 |
| Balanced RL | 55.34 ± 0.92 | 57.00 ± 1.19 |
| **Logics-SWE（RRE+MOPD）** | **58.04 ± 0.20** | **59.00 ± 0.47** |

相对 Base：Pro-618 **+5.40**，Multilingual **+2.78**。最终模型分项（Pro-618）：A **58.07**、B **59.37**、C **56.63**。Multilingual 上增益主要落在 A、B 与未路由组，C 均值可持平——作者也诚实写出「并非均匀铺开」。评测脚手架：Pro 用研究侧 R2E-Gym；Multilingual 用 SWE-agent——**跨脚手架绝对分不可直接比**，要比的是相对 Base / 联合 RL 的抬升与最弱类。

## 🔬 最有意思的部分：整合不是权值平均

三点工程味很重：

1. **同域内造分区**：跨域专家默认「域已可分」；仓级 SWE 没有现成边界，必须用可审计标签规则建立粒度——Labeler 本身是基础设施，不是附属脚本。
2. **类别切分 ≠ 更强专家**：初始类别 RL 会抬平均训练成功率，但仍有实例级 regress；RRE 显式巩固成功行为并重选任务，才是「专精」真正发生的地方。
3. **MOPD ≠ MoE 部署**：整合是 on-policy distillation，不是专家权值算术平均；推理单模型。奖励外推只保留相对参考「变好」的方向，避免把教师噪声整包灌进学生。

和同周 [CodeMidas](./20260922_CodeMidas_论文解读_只靠源码榨出五千可验证coding_RL环境.md) 并读：CodeMidas 解决「从源码规模化可验证环境」；本文解决「有了混合任务后，如何避免 see-saw、把专精收益收回一个策略」。环境飞轮与分布感知后训练是上下游，不是二选一。

## 🤔 我的判断

定位：**仓级 SWE 后训练的分布感知方法论文 + 可部署单模型**，测量协议（分项、min category lift、expert-gain recovery）与算法同等重要。亮点三：

1. 把 category see-saw 从「感觉」写成可复现对照（Pooled vs Balanced vs 专家线）；
2. RRE 让专家用**自己的**验证成功轨迹巩固，不靠外部解轨迹——闭环更干净；
3. MOPD 收回单模型，部署成本与联合 RL 同阶，却明显更高一档。

局限：Pro-618 是审计子集非官方全榜；A/B/C 是一种粗 Domain L1 操作分组，换标签规则可能改故事；公开 Env-2.5K 只是训练子集（2,553），完整语料与轨迹未全放；Multilingual 增益不均。对 Byron：若你在做 coding-agent RL / harness-dojo，可直接搬三件套——**(a)** 训练池先过可审计多轴标签，评测固定报「总分 + 每类 + 最弱类相对联合基线的 lift」；**(b)** 专家线用 RRE 式「刷新掌握度 → 自轨迹 Repair → 扩前沿」，别假设「切开就能专精」；**(c)** 整合优先 on-policy 多教师蒸馏而非永久 MoE；**(d)** 与 [CodeMidas](./20260922_CodeMidas_论文解读_只靠源码榨出五千可验证coding_RL环境.md) 并联造课表，与 [RuVerBench](./20260922_RuVerBench_论文解读_裁判验长轨迹Rubric仍有噪声.md) 对照：能执行验证的继续吃稀疏环境奖励，过程 rubric 另校准。

一句话收束：SWE 后训练若只盯 aggregate，你会庆祝一场掩盖了类别损失的胜利——CategoryAwareSWE 逼你先把「一个」拆明白，再负责任地合成回「一个」。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
