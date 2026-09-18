---
month: 202609
week: 3
date: 2026-09-18
type: 论文解读
slug: MoModels
---

# 模型池越大越强？异质 MAS 里「多」经常变成「更差」

你有没有过这种体验：Hugging Face 上几百万个模型，做 multi-agent 时下意识觉得「候选越多越好」——路由多几个、投票多几票、再塞个 LLM-as-a-Judge，总能比单模强。NVIDIA / 哥本哈根这篇 *Mo’ Models, Mo’ Problems*（arXiv:2609.17306）专门拆这件事：**在 before-generation（routing）与 after-generation（majority vote、LLM judge）两类 MAS 上，系统评估 8 种模型池选择策略**。看完我的感受是：标题不是段子——**扩大候选池常常把实际表现打到低于池内最强单模**；真正相对单模仍能站住的，往往是**同一模型家族内**的候选选择。

## 核心摘要

作者在 HLE、GPQA-Diamond、Frontier Science–Olympiad 等硬科学推理基准上，从 **23** 个 2024–2026 开源 LM（跨 **6** 个架构族、**2B–1.6T**、含 4 个科学特化模型）各采 **5** 代回答，用 gpt-oss-120b 当等价性裁判（与人标注子集约 **93%** 一致，Cohen’s κ **0.63**）。Oracle（理想选答）随 \(k\in\{3,5,10,15,20\}\) 单调变好，尤其按 accuracy / accuracy×diversity / LLM 建议组池时「理论天花板」最高；但 **Achieved MAS** 几乎相反——**候选越多，相对池内最优单模的 ∆MAS Gain 往往越负**。八种选池策略覆盖预评估信号（Size、Family、LLM Chosen）与后评估信号（Accuracy、IoU 正确答多样性、Error 多样性、以及 Accuracy×Err / Accuracy×IoU）。路由侧仅 **IoU** 与 **Family** 常见正增益；投票与 Judge 对异质池随 \(k\) 掉得更陡。同质多数票在 HLE 上可从 pass@1Best **29.4%** 提到 majority@5Best **32.2%**，Judge@5Best 到 **36.5%**，但异质 intentional 子集多数仍输给随机池或单模。结论写成工程口令：**别把任意异质模型往 MAS 里堆；模型选择本身是一等设计选择，且往往要先实测再组池。**

## 论文信息

- **标题**：Mo’ Models, Mo’ Problems: How to best select model pools when designing Multi-Agent Systems
- **作者**：Sara Vera Marjanović（University of Copenhagen；NVIDIA 实习期间完成）, Jiacheng Xu, Aleksandr Laptev, Grigor Nalbandyan, Erik Arakelyan, Evelina Bakhaturina（NVIDIA）
- **机构**：University of Copenhagen · NVIDIA
- **链接**：https://arxiv.org/abs/2609.17306 （2026 年 9 月 15 日提交）· 代码 https://github.com/spaidataiga/mo-models

---

## 🎯 为什么「候选列表」成了被忽视的 MAS 旋钮

MAS 通常被讲成结构故事：路由 vs 投票、级联 vs 辩论、有没有 orchestrator。作者指出另一旋钮常被默认掉——给定近乎无限的 \(M=\{m_1,\ldots\}\)，**子集 \(C\subset M\) 怎么选**？模型卡上有 size / 家族 / 偶尔有领域标签；校准集上又能算 accuracy、正确答 Jaccard 距离（IoU）、错误多样性。但「oracle 很高」≠「你的 router / vote / judge 真能吃到」。

对 harness / 模型池产品尤其相关：很多人用「再加一个强模」当免费增益；这篇用系统实验泼冷水——**异质多样性会污染答案空间**，尤其在需要塌缩成单一最终答的科学推理上。

## 🏗️ 方法：两种 MAS × 八种选池 × Oracle vs Achieved

**Before-generation**：按 AvengersPro 思路训简单聚类路由，优化训练集 pass@1。  
**After-generation**：

1. **Majority vote**：E5-Large 嵌入 5 代回答，凝聚聚类（默认最大余弦距离 **0.15**），取靠近质心的代表再判对错；
2. **LLM Judge**：GenSelect 式锦标赛，用 gpt-oss-120b，因上下文限制先 per-model 再每批 ≤8 模型比到唯一赢家。

选池策略（摘要）：

| 策略 | 信号来源 | 直觉 |
| --- | --- | --- |
| Size | 预评估 | 参数量排序，每族只取一个 |
| Family | 预评估 | 整族全收（Olmo / Llama / Qwen3 / Qwen3.5 / gemma / gpt-oss） |
| LLM Chosen | 预评估 | GPT-5 Deep Research 读模型描述选 top-k |
| Accuracy | 后评估 | 校准 pass@1 |
| IoU | 后评估 | 正确答集合多样性 |
| Error | 后评估 | 错题模式多样性（MCQ） |
| Acc×IoU / Acc×Err | 后评估 | 50–50 调和 |

主文报告相对「该池内最优单模」的 **∆MAS Gain**；绝对精度放附录。校准集约 **15.5k** 题，与测试相对模型排序高度相关（\(r>0.9\)）。

## 🧪 关键证据：Oracle 向上，Achieved 向下

**模型信号**：准的模型倾向于答对同一批题（Mantel \(r_M=0.931\)），但错误相关更弱（\(r_M=0.384\)）；size 与 accuracy 中等相关（Spearman \(r_s=0.583\)）。更刺的是 specialty 神话：相对 Llama-3.1-8B 基座微调的物理/化学特化模型，**通用 Llama 在含物理化学在内的各科学域上并不更弱**；特化模型「独有答对」也不集中在其招牌领域——**不能从训练数据标签外推正确答多样性**。

**Oracle（图 4）**：\(k\) 越大越好；Accuracy 系与 LLM Chosen 天花板最高；Error / IoU 预言提升最小——模式跨数据集守恒。Family 整体绝对分不高，但某些族（尤其 gemma）**预言相对单模增益**很诱人。

**Achieved（图 5）**：叙事翻转——**更多模型几乎总在伤相对增益**；多数 intentional 组还不如随机子集。跨三种 MAS，**同架构 Family 相对最稳**（常常是「损伤最小」而非大胜）。HLE 上几乎只有单族 MAS 相对基线单模仍为正。路由更吃 IoU 多样性；投票/Judge 随异质 \(k\) 掉得更狠。同质系统里投票/Judge 仍能涨（HLE：29.4% → 32.2% → 36.5%），异质 intentional 子集却常输随机。

## 🔬 最有意思的部分：「多样性太多」与 Judge 救不了堆料

讨论两句特别值得写进设计备忘：

1. **There can be too much diversity**：高解多样性污染答案空间，推理任务上更难隔离正确响应。成功用多样性的工作，往往是**大量 agent × 有限多样性**（改 system prompt、开工具、或只混很少基座），而不是任意异质模型大乱炖。
2. **Routing 不只需要正确答多样性**：IoU 路由相对最好，但很少足以稳定超过单模；专家必须在**任务分工**上可学，而不只是答对集合互补。
3. **更强 Judge / 中心编排或许能救异质，但不是免费的**——作者的简单 GenSelect 锦标赛并没有把 intentional 异质子集从「输随机」里捞上来（至少在 HLE 上）。

这和 Byron 关心的 LLM-as-Judge 闭环也咬合：裁判层解决不了「候选池先选错」的问题；Rubric 再漂亮，输入若是一堆互相拆台的异质输出，闸门只会更抖。

## 🤔 我的判断

定位：**模型池选择的测量论文**，不是新 MAS 架构。价值在于把「多模型 = 更强」从默认信念变成可证伪曲线，并给出可执行的优先序：

1. 先报 **相对池内最优单模** 的 ∆，不要只报绝对分；
2. 异质池默认有罪，直到实测；优先试 **同族 / 同质多实例**；
3. Oracle 高不能当上线依据——必须跑 Achieved；
4. 路由可显式优化正确答多样性；投票/Judge 更怕解空间被污染；
5. 领域微调标签 ≠ 可用的多样性信号。

局限：科学 QA 为主；MAS 结构刻意保持简单（无复杂工具交互）；Judge 单一后端可能偏置；HF 上千万模型的外推要谨慎。对 Byron：做 multi-agent / model pool / MCP 路由时，这篇该进必读——它直接打在「再挂几个模型」的产品冲动上。

一句话收束：下一个问题不是「还能加哪个 SOTA」，而是**你的 MAS 到底需要哪一种行为模式的互补**——答错了池，结构再花也是负增益。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
