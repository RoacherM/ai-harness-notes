---
month: 202610
week: 2
date: 2026-10-14
type: 论文解读
slug: LongHarness
---

# 证据和结论别埋在对话历史里：共进化状态，才把一亿 token 语料问对

你有没有过这种体验：让 Claude Code 或 Codex 在一大堆财报里算一个跨文件的数，它很快找到了表，然后开始一页页往回翻；第 160 多步时它自己发现「5 月那个值应该是 4400 不是 4100」，嘴上说了要改，最后交出来的还是用 4100 算的结果。你心里清楚：**证据它其实看到了，结论却没跟上。**

复旦与上海人工智能实验室等单位的 **LongHarness**（arXiv:2610.13075）把这种失败拆成两种互相强化的模式：**unbounded exploration**（检索不断扩张历史却不解决和决策相关的信息缺口）和 **evidence-state drift**（已获取的证据或后来的更正没能进入当前结论）。它的解法是把「证据」和「结论」从交互历史里拎出来，做成两份耦合的持久状态。看完我的感受：这是一个不改权重、只改 harness 的工作，却在一亿 token 级的 OfficeQA Pro 上，用同一个 Flash 级 backbone 把 Claude Code 和 Codex 甩开了两位数。

## 核心摘要

LongHarness 是一个模型无关的单 agent harness，状态写作 $S_t=(E_t,C_t,H_t)$：**EvidenceBank** 存从已读段落中抽出的、带源引用的陈述；**Conclusion State** 存已确立的论断、受支持的假设与未解决的缺口；$H_t$ 才是原始交互历史。核心机制 **Evidence–Conclusion Coevolution** 让未解决的结论缺口指挥下一步搜什么、读什么，新证据入库后再回头强化或修订结论；**Evidence Lifecycle Management** 规定证据怎么进、怎么留、怎么退：每条候选陈述必须引用此前 read 动作登记过的源，修正不能原地改，而是新建 active 条目并把旧条目标为 superseded，撤回则标 retracted，被取代或撤回的条目不能再被 compute 或 final 引用。在 OfficeQA Pro（V1 约一亿 token、697 份文档；V2 扩到 1,435 份并加入复合统计题）上，GLM-5.3-Flash 直接推理 exact match 只有 **15.8% / 4.4%**，挂上 LongHarness 后到 **60.2% / 61.1%**；同 backbone 下比 Claude Code 在 V1 高 **12.0** 点、比 Codex 在 V2 高 **18.9** 点。反直觉的是 V2 更难，LongHarness 的成绩却没掉，而且中位交互轮数只从 22 涨到 30，Claude Code 则从 21 涨到 116。价值定位：它给「长上下文推理」换了一个视角，把核心推理过程看成证据生命周期管理，而不是更长的窗口或更多的检索。

## 论文信息

- **标题**：LongHarness: Advancing Agentic Reasoning over 100M-Token Corpora via Stateful Evidence Management
- **作者**：Songtao Huang, Zhijie Zhong, Xiaohan He, Shuaiyu Zhang, Zhiyin Yu, Li Kang, Heng Zhou, Yutao Fan, Xiangchao Yan, Bo Zhang, Shiyang Feng（通讯）
- **机构**：Fudan University · Shanghai Artificial Intelligence Laboratory · Peking University · Shanghai Jiao Tong University · USTC · Harbin Institute of Technology
- **仓库**：论文未给出代码链接
- **链接**：https://arxiv.org/abs/2610.13075 （2026-10-09）

---

## 🎯 为什么这件事值得写

作者把语料级长上下文推理分成三层：**L1 检索**（找到相关段落、过滤干扰）、**L2 证据编辑**（选、对齐、修订）、**L3 转化**（把证据变成答案）。训练侧在监督检索轨迹和记忆编辑，harness 侧在做自适应文档访问和持久存储，但「证据积累 → 结论演化」这一步通常是隐式的，藏在会话历史、自由笔记或一次性推理里。

| 路线 | 证据放在哪 | 结论放在哪 | 典型失败 |
| --- | --- | --- | --- |
| 直接推理（BM25 + 256K 窗口） | 预选段落 | 单次推理 | 预选漏掉中间结论需要的信息 |
| Claude Code / Codex / RLM | shell 输出与历史 | 历史与临时推理 | 无界探索、更正没进最终计算 |
| 通用 working notes | 自由文本 | 自由文本 | 结构弱，难追踪缺口 |
| **LongHarness** | EvidenceBank（源链接 + 生命周期） | Conclusion State（论断 / 假设 / 缺口） | 计算校验不保证操作数与源一致 |

这和 [ReHarness](./20261012_ReHarness_论文解读_反思写进记忆就不再改_反向修订让harness状态也能吃到下游反馈.md) 的「反向修订」、[DerivAudit](../week1/20261001_DerivAudit_论文解读_记忆不是摘抄是推导_分布式证据悖论卡死写入闸.md) 的「记忆是推导不是摘抄」以及今天的 [PANINI-DR](./20261014_PANINI-DR_论文解读_深研别把协调和研究搅在一条链里_共享工作区加验证闸让难题答对近翻倍.md) 是同一个家族：**状态必须显式，且要能被后来的证据改写。**

## 🏗️ 机制：两份持久状态 + 一张小动作表

推理周期上，两份状态这样共进化：

$$
E^{+} = A_\theta(q, D, E, C),\qquad C^{+} = R_\theta(q, E^{+}, C)
$$

$A_\theta$ 是在当前结论指导下的探索与入库，$R_\theta$ 是用 active 证据修订结论；两者由同一个策略交替实现。

**结论指导探索。** 对每个论断，agent 先想清楚「什么观察能确立、限定或推翻它」，再据此写 query、选阅读范围；竞争假设驱动能区分它们的搜索；缺口指向缺失前提。观察没能填上缺口时，改写 query 或扩大阅读范围，但仍拴在推进结论所需的信息上。

**证据驱动修订。** 观察到 EvidenceBank 更新结果后，再逐条判断论断是否仍成立、需要限定还是必须改；提交前检查剩余缺口会不会改变所要的决策。

**动作空间（Table 5/6）。**

| 操作 | 作用 |
| --- | --- |
| search | 按信息需求定位文档或段落 |
| read | 有界阅读并登记源引用 |
| upsert | 批量提交源链接陈述，逐条校验，可部分接受 |
| revise | 用新陈述取代旧条目，或以 null 撤回 |
| compute | 对证据值做确定性数值计算，校验声明的证据 ID |
| final | 提交答案，引用必须解析到 active 证据与源段落 |

控制器负责校验结构、引用登记与预算；陈述是否忠实于源，仍由策略判断。预算设置为 200 步、60 次搜索、累计 100 万 token 阅读、单次读 16,384 token；历史观察完整保留，不做摘要替换。

## 🧪 关键证据

**主结果（Table 1，Exact %）。**

| Backbone | Harness | OfficeQA V1 | OfficeQA V2 | AA-LCR | LongBench v2 Long | Avg. |
| --- | --- | --- | --- | --- | --- | --- |
| GLM-5.3-Flash | 直接推理 | 15.8 | 4.4 | 80.0 | 62.0 | 40.6 |
| GLM-5.3-Flash | Codex | 46.6 | 42.2 | 77.0 | 69.4 | 58.8 |
| GLM-5.3-Flash | Claude Code | 48.1 | 35.6 | 79.0 | 63.0 | 56.4 |
| GLM-5.3-Flash | RLM | 45.1 | 38.9 | 56.0 | 60.2 | 50.0 |
| GLM-5.3-Flash | **LongHarness** | **60.2** | **61.1** | **84.0** | **75.9** | **70.3** |
| Qwen3.8-Flash | Codex | 49.6 | 50.0 | 81.0 | 69.4 | 62.5 |
| Qwen3.8-Flash | Claude Code | 53.4 | 41.1 | 83.0 | 63.9 | 60.3 |
| Qwen3.8-Flash | **LongHarness** | **61.7** | **51.1** | **87.0** | **76.9** | **69.2** |

直接推理一侧，即便是 Kimi-K3、GPT-5.6-Sol 这类强模型，V1 也最多 23.2%、V2 最多 5.6%。

**消融（Table 2，GLM-5.3-Flash，Exact %）。**

| 变体 | V1 | V2 |
| --- | --- | --- |
| 完整 LongHarness | 60.2 | 61.1 |
| 去掉两份持久状态（只留历史） | 55.6 | 53.3 |
| 去掉 Conclusion State 与检索反馈 | 54.9 | 51.7 |
| 证据只能追加（Naive Accumulation） | 56.4 | 51.1 |
| 去掉「用缺口指导检索」的指令 | 55.6 | 53.3 |
| 结构化结论换成可编辑自由笔记 | 53.4 | 53.3 |

五个消融在 V2 上的损失全部大于 V1，只能追加的证据库在 V2 掉 10.0 点，最狠。

**三次重复（Table 9，GLM-5.3-Flash）。** OfficeQA V2 均值：LongHarness 59.63（std 1.28），Codex 41.11（4.00），Claude Code 32.59（4.21），RLM 36.30（4.49）。LongHarness 不仅更高，也更稳。

**泼冷水。** 主表每个设置只跑一次；Qwen3.8-Flash 在 V2 上只比 Codex 高 1.1 点，增益随 backbone 变化明显；阶段审计用 GPT-5.6-Luna 判，带模型裁判噪声；运行时不会在证据失效后自动改写依赖它的结论，compute 也不校验字面操作数是否等于源值。

## 🔬 最有意思的部分

1. **一个算例讲透 evidence-state drift。** 80 个月度支出的几何平均：Codex 170 步，前 22 步就找到表，随后连续 103 次往回读；它用 4100 算出 4958.09，后来认出 4400 才对，最终答案却没改。LongHarness 10 个策略动作：把 4400 写进 EvidenceBank，把「跨期一致性」记成缺口，进一步阅读既确认了它又修订了另外两条，最后算出参考答案 4962.46。**Codex 输在没有把更正带进最终计算，而不是没看到证据。**
2. **轮数曲线比准确率更说明问题。** V1→V2 中位轮数：LongHarness 22→30，Codex 19→43，Claude Code 21→116；Codex 在 V2 有 21.1% 的题耗尽轮数预算。缺口显式化之后，agent 不必反复重找已确立的值。
3. **三层阶段率拆出了收益来源。** V2 上 LongHarness 的 L1/L2/L3 为 87.8 / 87.3 / 79.7，Claude Code 是 72.2 / 84.6 / 58.2；V1 的 L1 两者都是 85.0，差距主要在后两层。**同样找到证据，能不能用上才是分水岭。**
4. **自由笔记比结构化结论差。** 给 agent 一份能写结论、问题和引用的可编辑笔记，V1 反而掉 6.8 点。「可以记」和「被结构逼着记」不是一回事。

## 🤔 我的判断

定位：**面向语料级长上下文推理的 harness 状态设计**，属于不训练、只改运行时协议的工程方法突破，也给 L1/L2/L3 分层诊断提供了可复用的度量框架。

亮点三条：

1. 同 backbone 下对 Claude Code、Codex 有两位数优势，且三次重复方差明显更小；
2. 消融把「持久状态」「结论反馈」「可修订证据」「缺口指导检索」「结构化 vs 自由笔记」逐一拆开，每一项都有贡献；
3. superseded / retracted 的生命周期语义清楚，final 只能引用 active 证据，是一个能直接抄的合同。

问题与局限：主结果单次运行；增益依赖 backbone；没有开源代码链接；结论不会随证据失效自动重写，正确性仍押在策略自觉上；阶段审计依赖 LLM 裁判。

对工程的直接启发：**(1)** 长任务 harness 里把证据库和结论状态分成两份结构化对象，别只靠历史；**(2)** 证据修正走「新建 + 标 superseded」，不原地覆盖，final 和 compute 只认 active；**(3)** 把「未解决缺口」显式写下来，并在 prompt 里要求用缺口指导检索，光给状态不够；**(4)** 评测时按 L1/L2/L3 拆失败，比只看最终 EM 更能定位问题；**(5)** 和 [Harness-Design](../../202609/week3/20260918_Harness-Design_论文解读_脚手架不是一整块_组件要按模型和预算配.md) 的结论对得上：通用 coding agent 脚手架不等于特定任务的最优 harness，同一个 Flash 模型换一套对路的状态设计，就能拉开两位数。

一句话收束：**LongHarness 证明，一亿 token 级的问答不是窗口不够大，而是证据和结论没人管；把它们做成会共进化、可追溯、能撤回的状态，Flash 模型也能问对。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
