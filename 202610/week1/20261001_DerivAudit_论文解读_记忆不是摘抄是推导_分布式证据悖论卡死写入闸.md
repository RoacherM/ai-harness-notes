---
month: 202610
week: 1
date: 2026-10-01
type: 论文解读
slug: DerivAudit
---

# 记忆不是摘抄是推导：分布式证据悖论，把「看起来有出处」的写入卡在闸门上

你有没有过这种体验：长程 agent 把对话压成一条 memory，后面任务当事实用；出了事回头查 citation，要么「引用不全但历史其实支持」，要么「每条原料都在，拼出来的因果/时态却从未成立」。NYU 的 Hongjun Liu、Chen Zhao 这篇 **Memory Is a Derivation**（arXiv:2609.36130）把持久记忆写成**从交互史到可复用前提的推导**，并用框架 **DerivAudit** 审计写时是否具备语义推导完整性。看完我的感受：这正好打在 Byron 的 **long-horizon / agent memory · judge/evals** 线上——和「检索是否召回」不同，它问的是**写进去的那一刻，历史是否许可这句话的全部含义**。

## 核心摘要

持久记忆一旦写入，后续任务常不再回看原始交互，还会继续组合记忆推新结论——因此写时可靠性是边界。作者提出三个耦合要求：**证据范围**（支持可能超出 writer 附带 citation）、**组合有效性**（单点事实成立 ≠ 关系/时态/事件状态成立）、**准入可靠性**（既要拦住无支持写入，又不能误杀后来需要的有效记忆）。DerivAudit 对同一候选记忆做 citation-only vs 扩展写前史（BM25 最多 +12、总 cap 16）对照审计，并把记忆拆成必须全部成立的 **semantic obligations**。在 LoCoMo 与 HaluMem-Medium 共 **400** 条未编辑写入上：citation 不足的记忆里约 **60%** 可被更广历史「修复」为支持，但仍有约 **17–21%** 扩展后仍无支持；无支持记忆在多种验证模型下仍常被准入（约 **59–89%**）。受控实验揭示 **distributed-evidence paradox**：有效记忆常要跨交互拼证据，但多个「各自像真」的前提会让无支持组合更难查；逼验证器多查几条**已知为真**的义务，还会因 **accumulated verification noise** 误拒有效记忆。下游探针：扭曲记忆与缺失有效记忆都会大幅抬高后续失败。

## 论文信息

- **标题**：Memory Is a Derivation: The Distributed-Evidence Paradox in Long-Term Agents
- **作者**：Hongjun Liu、Chen Zhao
- **机构**：New York University
- **链接**：https://arxiv.org/abs/2609.36130 · https://arxiv.org/pdf/2609.36130
- **观察时间**：2026-10-01（长程 agent 记忆写时审计 / DerivAudit）

---

## 🎯 为什么这件事值得写

Mem0 / MemGPT / A-mem 等把「形成–演化–检索」做成标配；Eywa、ConsistencyGate、MemTxn、MemIR 开始把记忆当可靠性边界。仍缺一块：**writer 给的 provenance 可能漏掉真支持，而单点可支持的事实拼出的关系可能完全非法**。事实性细粒度工作（FActScore、FactGraph、QASem 等）多盯生成文本；DerivAudit 把对象换成**会被后续当前提用的持久状态**，并同时量「证据在哪 / 组合加了什么含义 / 闸门留了什么」。对 harness：这是 memory 写入旁路的 **judge 规格**——不是再加一个「看起来合理」的 LLM 点头。

## 🏗️ 机制：DerivAudit 三问

**证据范围。** 同一写前史两种视图：仅 citation vs citation∪检索扩展；排除写后交互与「用已写入记忆证已写入记忆」。provenance-repaired = citation 不足但扩展后支持。检索触顶标 unresolved，不武断算历史无支持。

**组合有效性。** 记忆 \(m_i\) 分解为义务集 \(\mathcal{O}(m_i)\)；全员 Supported 才算记忆支持。参考标签由两模型独立答 D/E/R 原语再路由（supported / contradicted / insufficient / ambiguous），分歧盲审。验证视图对比 holistic、atomic claims、predicate–argument QA，以及 compact-relation / composition-graph / per-obligation 等显式关系表示。另有 70 组 matched family：局部 vs 分布式 × 有效 vs 无效，固定候选与真值。

**准入可靠性。** 重放 citation-only holistic、扩展史 holistic、扩展史 obligation-aware；量 supported retention 与 unsupported admission，并单列 repaired。另测「再加 2/4 条已知为真的义务」是否拖垮分数与准入。下游：固定检索位与上下文，只改正确 / 扭曲 / 缺失记忆，估伤害与拒真代价。

## 🧪 关键证据

**自然写入（400 / 准入 391）。** LoCoMo / HaluMem citation 不足率约 42.5% / 51.0%；扩展史修复其中约 60.0% / 59.8%，整体支持率分别升至约 79.5% / 77.0%。Qwen 上 repaired retention：citation-only 82.1% → 扩展 93.8%（+11.6 pp，\(p=0.007\)）；obligation-aware 92.9%。但 unsupported admission：Qwen 84.6→76.9（\(p=0.24\) 未稳），Gemma 可到 59.0，Llama 甚至 88.5——**扩史稳提真、不稳降假**。

**组合与悖论。** 512 条受控套件上，atomic 对无支持关系检测极弱（Qwen 16.5%），PA-QA 检测升却多跨段有效记忆留存差；compact-relation 在 Qwen 上检测约 59.5%、多跨段留存 94.9%。分布式前提下，holistic / obligation-aware 对无效例检测可低至约 8.6% / 11.4%。机制对照：删光许可证据几乎必拒；把「像真」前提换成等长无关内容，假接受从约 65–88% 掉到 0–10%——悖论主因是**看似合理的前提组合偏置**，不是单纯上下文变长。

**验证噪声。** 有效记忆上再挂 4 条真义务：Qwen 中位 log-odds −7.7（98.7% 下行）；Gemma 准入 93.3%→2.6%，Llama 97.4%→35.5%。查得越细，越可能误杀真记忆。

**下游。** 扭曲记忆相对正确记忆抬失败约 +83.8–93.0 pp；拿掉有效记忆约 +92.8–99.5 pp。自然例：志向「想建体育学院」+「社交圈支持」被写成「正在建」且「圈子支持该事业」——事件状态与关系双重越权。

**泼冷水。** 主标签仍是模型互评（分层人工一致约 96%）；检索有界；自然数据里明确矛盾与关系型义务偏少；骨干与视图差异大；巩固漂移为观测相关非严格因果。

## 🔬 最有意思的部分

1. **Citation 匹配 ≠ 历史许可。** 有记录 citation 过关、扩展审计仍判无支持（跨实体时长错绑等）——只做 provenance 门禁会漏组合幻觉。
2. **分布式证据悖论是真张力。** 系统需要跨段拼真，验证器却在「多段像真前提」下更易放行假关系——这不是调阈值能一键消掉的。
3. **Accumulated verification noise。** 「多查一点更安全」在有效记忆上可反向；judge 设计必须管**义务粒度与聚合规则**，不是义务清单越长越好。
4. **拒真与纳假对称伤下游。** 缺失与扭曲都接近「换错前提」量级——记忆闸门不能只优化一边的精确率。
5. **与写时后悔 / 外部记忆文献对齐。** 本仓库周前 DSSR、EMem-Bench、JITMem 分别打「写时后悔」「外部记忆增益」「读时策展」；DerivAudit 补的是**写时语义许可**这一刀。

## 🤔 我的判断

定位：**长程 agent 记忆写时的推导完整性框架与实证**（测量 / 评测基础设施，附受控机制实验）。亮点三：

1. 把记忆明确成 derivation，三个审计问题可操作；
2. 分布式证据悖论与验证噪声给出可复述的失败模式；
3. 下游因果探针证明写时对错会变成行为对错。

局限：标签与检索近似；跨模型不稳；尚未闭合「在线写–用–再写」全系统。对 Byron：**(1)** 记忆写入旁路的 judge 至少两视图：citation vs 写前检索扩展，并单独报 provenance-repaired；**(2)** 义务要显式含关系 / 时态 / 事件状态，禁止「原子事实全绿就放行」；**(3)** 准入指标必须双报 retention 与 unsupported admission，并切片 repaired；**(4)** 义务堆叠要防噪声——优先少量高风险义务 + 可校准阈值，而不是无限细拆；**(5)** 与 MMP 外部记忆扩展对齐时，把「语义推导完整性」写进 Manifest 级规则，而不是只靠检索分数。接 [DSSR](../202609/week5/20260929_DSSR_论文解读_预算够了仍忘_写时后悔才是常驻记忆瓶颈.md) / [JITMem](../202609/week4/20260925_JITMem_论文解读_记忆别在写入时裁剪_读时按任务现场策展同一条轨迹能榨出不同课.md)：DerivAudit 补的是**写闸上的组合语义，而不只是容量或读时策展**。

一句话收束：**记忆一旦当前提，就不再是摘要——DerivAudit 要问的是：写前历史，是否推导得出你准备永久相信的那句话。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
