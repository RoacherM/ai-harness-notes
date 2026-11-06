---
month: 202611
week: 1
date: 2026-11-06
type: 论文解读
slug: Drex
---

# 决策别再靠「生成再解析」：Drex Max 在 Decision Index 0.4.1 公共套件拿到 67.52，压过 Jev 1.14.0 的 60.70，典型请求 183 ms

你有没有过这种体验：路由、验收、护栏这类题，选项是调用方现场给的，不是训练时焊死的标签头。你让大模型长链推理再吐一个选项名，又贵又慢，还要解析；选项一换序，答案偶尔翻脸。Jev 把这条路收成 System One：给 state 和一组 typed 问题，直接回选项上的概率分布。问题变成：**开源决策模型能不能在真实决策榜上顶住前沿生成模型，同时稳住校准、顺序鲁棒和时延？**

Nace.AI 的技术报告 **Towards Foundation Decision Models: Drex** 交出四个开源决策模型：DLM 8B、v1.5 9B、v2.0 35B-A3B、Max 27B 多模态。Max 在 Decision Index 0.4.1 公共套件 chance-corrected skill 到 **67.52**（Jev 1.14.0 为 60.70），典型请求中位时延约 **183 ms**（对照 GPT-6 Sol ~2.0 s、Claude Opus 5.5 ~3.5 s）。看完我的感受是：**这是把「用户现场给选项」的判别接口，当成一类要单独建的基础模型，而不是 LLM 的副业提示词。**

## 核心摘要

决策模型接收一段 state（文档、记录或 agent 上下文，Max 可带图）和若干问题；每题是多选、是否、或有序量表，返回选项上的概率，不做自回归解码。Pointer 系（DLM / v1.5 / v2.0）对上下文编码一次后跨题共享；Max 基于 Qwen3.8-27B，去掉因果掩码的全注意力，用无偏置线性 readout 把头状态映到最多 255 个选项码 logit，每题重编码上下文换精度。训练主菜是 SFT（Max 另加选项置换与 state 字段打乱的一致性损失）+ **RLAF**（reinforcement learning from adjusted feedback：用可验证结果与 LRM-as-a-judge 概率构造调整目标，再相对 SFT 检查点做 KL 约束），Max 还有视觉 SFT 阶段。消融：RLAF 让 Decision Index 大约 +2.0；一致性项把 flip rate 从 6.3% 压到 4.2%；视觉阶段不伤文本。公开权重与推理栈在 https://github.com/nace-ai/drex-decision-models（Jev 兼容 / System One）。对工程的价值在于：同一接口四条骨干、成规模校准与顺序敏感性报告，以及「生成裁判 vs 单次前向决策」的成本对照。

## 论文信息

- **标题**：Towards Foundation Decision Models: Drex
- **作者**：Nischay Dhankhar、Alisher Amantay、Zhanibek Datbayev、Darkhan Akhmetov、Yash Shah、Nurmukhamed Ubaidullayev、Raman Saparkhan、Dos Baha、Abulhair Saparov（Nace.AI Research Team）
- **机构**：Nace.AI
- **链接**：https://arxiv.org/abs/2611.07854 （2026-11-05 提交；HF Daily Papers 2026-11-06）
- **代码 / 权重**：https://github.com/nace-ai/drex-decision-models · https://huggingface.co/nace-ai
- **产品页**：https://nace.ai/drex

---

## 🎯 为什么这件事值得写

分类头、打分头、嵌入模型的输出形状在部署前就定死了。Agent 路由与护栏却要：**上下文变、选项集变、还要校准概率。** 自回归生成再解析，或按 token 约束解码再拼概率，都会和「选项集合上的分布」错位，还贵。

Jev 把 System One 接口立住之后，缺的是可复现的开源族与训练配方。Drex 报告的贡献不在又一个排行榜截图，而在：

1. 同一请求格式打穿四条骨干（含扩散 LM、MoE、稠密多模态）。
2. 把 RLAF、一致性损失、视觉阶段拆开量贡献。
3. 在 Decision Index / Decision Bench / JevBench / DecideBench 上做成对统计比较，并公开顺序翻转与校准。

对盯着 Jev、Decision-1、erase-and-check 的人，这是目前最完整的「开源决策模型说明书」之一。

## 🏗️ 机制：System One + RLAF

### 接口

一次请求 = state + 若干 typed questions（choice / yes-no / ordinal score）。模型直接回每个选项的概率。Pointer 模型一次编码 state、多题共享；Max 每题重编码，readout 到 A/B/… 码。

### 训练（Max 三阶段）

1. **SFT**：监督决策；另加选项置换一致性、state 字段/句子打乱一致性（因为 readout 架构本身不保证序无关）。
2. **RLAF**：对每题构造调整后的目标分布 q̃（可验证对错 + judge 概率），最大化带 KL-to-SFT 的奖励；比 one-hot CE 更吃得下「软标签 + 核实结果」。
3. **Vision SFT**：训图像编码器与投影，教模型看图做决策；报告称文本精度不掉。

Pointer 系走同一接口上的 Kev 式指针头与对应 RLAF 变体；v2.0 在 SFT 与 RLAF 之间还有任务特化阶段。

## 🧪 关键证据

Decision Index 0.4.1（chance-corrected skill，公开套件）：

| 模型 | Public suite |
| --- | --- |
| **Drex Max** | **67.52** |
| Drex v2.0 | 61.25 |
| Drex v1.5 | 52.76 |
| Drex DLM | 45.78 |
| Jev 1.14.0 | 60.70 |
| GPT-6 Luna Decisions | 52.31 |

Artificial Analysis 独立 Decision Index 上 Max 在决策模型里第二（64，三强为 65）。Decision Bench v4：Max 超过 Jev 1.14 与自家小模型，与 d3-turbo、GPT-6 Sol 统计打平，落后 Claude Opus 5.5 约 1.6 分；DecideBench 上决策模型里最高。时延：Max 典型请求约 183 ms。校准在约 23 万决策上报告；顺序实验（原序 / 逆序 / 随机置换）显示 top 翻转多发生在选项多且置信低的题，按「会翻转就拒答」并不比按低置信拒答更准。

## 🔬 最有意思的部分

**RLAF 不是换皮 RLHF。** 它显式用核实结果去改写目标分布，再用 KL 拴住 SFT，消融里 +2.0 Decision Index 说得过去。一致性损失专治 readout 的序敏感，flip rate 6.3%→4.2%，是工程上能抄的小开关。另一边，Max 每题重编码换精度，指针模型更吃「一长 state、多题」的批价；报告把这条权衡写清楚，而不是只晒旗舰分。

## 🤔 我的判断

定位：**面向生产接口的开源决策模型族 + 可拆开的训练与评测说明书**，直接对标 Jev 生态。

亮点三条：

1. 公共 Decision Index 上明确超过 Jev 1.14.0，同时把时延打到百毫秒级。
2. RLAF + 一致性损失的配方和消融写得够具体，能当复现清单。
3. 四骨干共接口，方便在 harness 里当 System One 替身做路由 / 验收 / 护栏。

问题与局限：Max 与顶尖生成模型在部分 bench 仍有缺口；RLAF 依赖 judge 与核实信号的质量；许可证是 Open RAIL-M（DLM 骨干 CC BY-NC），商用要读条款。对工程的直接启发：**agent 里凡是「选项集由调用方决定」的门，优先试单次前向决策模型，把生成模型留给真要写字的步骤；验收时同时看准确率、校准、flip rate 和 p95 时延。**

一句话：Drex 把决策从「生成再解析」拉回「一次前向的概率接口」，并且终于有一份开源成绩单压过了 Jev 主线版本。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
