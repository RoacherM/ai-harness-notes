---
month: 202609
week: 4
date: 2026-09-25
type: 论文解读
slug: Qwen-Planner-Agent
---

# 模型练完还不够：Skills / Memory / Harness 要和权重一起共进化

你有没有过这种体验：coding-agent / 手机规划 agent 把权重刷到不错，上线却发现 **Skills 路由、Memory 注入、工具面编排** 才是成败分水岭——改一点 harness 指令，同一条 checkpoint 的 Tool Use / Skills 分就能跳一截；可工程里常见做法还是「模型训完 → 脚手架另起炉灶」，两边用不同反馈、不同版本节奏，失败轨迹也很少回流成 **可审查的共进化回合**。阿里通义 MAI / Token Hub 这篇 Qwen-Planner-Agent（arXiv:2609.29892）今天上 HF Daily Papers，核心不是又一个 mobile GUI agent，而是把 **AI for Data → AI for Training → AI for Harness** 收成闭环，并明确把 **Model–Harness Co-evolution** 写成 **受治理的开发路径**（reviewed / versioned offline，serving 时模型冻结）——作者自己也说，这是 pathway，**不是已经宣称实现了完全自治的持续共进化**。看完我的感受：和 Byron 关心的 [Meta-Harness](../week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) / [Harness-Zero](./20260923_Harness-Zero_论文解读_把专用Harness行为烤进权重_拆掉仍能超过挂着它的基线.md) / [SkillPivot](./20260925_SkillPivot_论文解读_失败别整轨反思_先找偏航点再对比后缀改技能.md) / MCP 工具面同频——**权重与脚手架必须共用同一套 action-feedback-verification 合同**，否则再强的 planner 也只是半成品。

## 核心摘要

Qwen-Planner-Agent 把 mobile planner 做成「Planner Model + 统一 Harness」的完整系统。任务形式化：\(c_t=\mathrm{H}_\eta(x,m_t,\mathcal{K}_t,o_{\le t},a_{<t})\)，\(a_t\sim\pi_\theta(\cdot\mid c_t,\mathcal{A}_t)\)——Harness 在请求时编排 memory、skills、工具与约束，再把结构化动作反馈与失败痕迹回流。三条 AI-for-AI 腿：（i）**AI for Data**——人闸门的 agentic data flywheel（构任务、采轨迹、清洗平衡、训练反馈→下一轮数据）；（ii）**AI for Training**——planning cold start + 混合环境 online agentic RL，并用 **CARE**（Competence-Aware Reward-and-Advantage Engineering）按组成功率切 progress / consolidation / efficiency 三档奖励，成功饱和后压推理与工具成本；（iii）**AI for Harness**——runtime 管 Skills/Memory/工具，离线用执行证据驱动模型与 harness 协同修订。MobilePA-Bench（1700+ 可执行任务）上 **27B Agent Overall 77.05%**，在评测集合里最高；固定 27B checkpoint 时 Harness 把 Overall **71.90%→77.05%**。另用独立 27B 基线做共进化：MobilePA-Internal **82.67%→88.50%**（4 轮，+5.83），MCPMark **38.00%→46.98%**（3 轮，+8.98），优于仅加 Harness 的 84.23% / 42.26%。非 mobile agentic 榜也有提升，通用能力大体保留，性能–成本曲线相对商业模型有利。

## 论文信息

- **标题**：Qwen-Planner-Agent: A Closed-Loop AI-for-AI Framework for Real-World Mobile Planner Agents
- **作者 / 团队**：MAI Team, Alibaba Token Hub, Alibaba Group
- **项目页**：https://tongyi-mai.github.io/Qwen-Planner-Agent/
- **链接**：https://arxiv.org/abs/2609.29892
- **观察时间**：2026-09-25（HF Daily Papers）

---

## 🎯 为什么这件事值得写

多数「agent 报告」要么只刷权重，要么只晒脚手架；把 **数据飞轮、训练目标、runtime harness** 接到同一条验证合同上，并公开承认 **共进化是受治理的开发路径而非已完成的自治神话**，在工程叙事里仍然稀缺。Mobile 规划又卡在长程可靠性与真机交互成本之间——正适合当 AI-for-AI 的压力测试。

对 Byron（skills / harness / MCP / coding-agent）：这篇把「Harness 不是提示词补丁」钉死——固定权重时 Skills/Memory/反馈就能抬分；交替训练与 harness 修订又能再抬一截，且 **MCPMark** 说明收益不止 mobile。对照 [EvoHarnessBench](../week3/20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) 的「脚手架越加越忘」，这里用版本化审查与 evolve-set 排除直训，是在防同一类过拟合。

## 🏗️ 机制：Data → Training(CARE) → Harness 共进化

**任务与合同**：环境后端（程序化沙箱 / LLM 模拟 / 精选真机）对 policy 暴露统一的动作–观测–验证接口；verifier 用历史与执行证据判完成。动作含 typed tool、memory 读写、skill 加载、澄清/拒绝与完成声明——偏工具合同而非像素 GUI。

**AI for Data**：专用 agent 构可执行任务、采交互轨迹、清洗过滤重加权；dev-set 上的 Tool Use / Memory / Skills / Sub-agent 与失败分布诊断，指导下一轮任务生成与采样。人闸门在关键处保留，避免飞轮失控。

**CARE**：对每任务采 \(G\) 条轨迹，组成功率 \(s\) 作能力信号。三档奖励：低 \(s\) 加 progress shaping；中档只保 outcome consolidation；高档加效率惩罚压 token/工具。另用 quality-preserving advantage calibration：效率档用 \(\sigma_{\mathrm{anchor}}=\sqrt{p_{\mathrm{high}}(1-p_{\mathrm{high}})}\) 作归一化分母下界，避免「全成功组里微小效率差被 GRPO 放大到与成败组同量级」。有界 LLM controller 每 \(N\) 步根据训练统计与 held-out 调 \(\{\lambda_{\mathrm{prog}},\lambda_{\mathrm{eff}},p_{\mathrm{low}},p_{\mathrm{high}}\}\)。同设置下 full CARE 与 Vanilla RL 精度相当，输出 token 约少 **32.5%**。

**AI for Harness / 共进化**：在线 Harness 组装 Skills、持久 Memory、工具与约束；离线把成功轨迹、失败结局与结构化反馈送入模型 RL 与 harness 指令修订。更新 **审查、版本化、离线生效**；serving 时 \(\theta\) 冻结。作者明确：这是 governed pathway，不是已验证的持续自治共进化。

## 🧪 关键证据

**MobilePA-Bench（Table 1，节选）**

| 系统 | Size | Overall | Tool Use | Memory | Skills | Sub-agent |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| GPT 6 Astra | Closed | 76.84 | 75.71 | 74.73 | 93.25 | 53.93 |
| Claude Opus 5 | Closed | 75.71 | 77.60 | 71.81 | 83.00 | 59.55 |
| Qwen Baseline | 27B | 67.22 | 68.37 | 67.82 | 73.75 | 47.19 |
| Qwen-Planner-Model | 27B | 71.90 | 72.79 | 70.74 | 79.25 | 55.06 |
| **Qwen-Planner-Agent** | **27B** | **77.05** | **77.79** | **74.76** | **86.25** | **59.55** |

固定 27B checkpoint：Harness 单独贡献 Overall **+5.15**；Tool Use **72.79→77.79**，Skills **79.25→86.25**。35B-A3B：Overall **64.79→69.91**（Sub-agent 不变 52.81）。相对 Qwen baseline，27B 全系统 **67.22→77.05**。估计输出成本约 **$2.41 / 1K tasks**（仅 output token，含 thinking；不含输入/工具/真机）。

**Model–Harness Co-evolution（Table 2，独立 27B 基线）**

| 方法 | MobilePA-Internal Overall | MCPMark |
| --- | ---: | ---: |
| Model Only | 82.67 | 38.00 |
| Model+Harness | 84.23 | 42.26 |
| **Co-evolution** | **88.50**（4 iters） | **46.98**（3 iters） |

共进化相对 model-only：**+5.83 / +8.98**；也高于 harness-only。evolve-set 只作开发反馈、不直训——比「在测试集上共进化」诚实一档。长历史 Memory（BEAM 等）上，同模型加 Harness 在 BEAM-500K/1M/10M 大幅抬分（如 27B Model：42.01→72.20 / 40.12→73.71 / 21.99→67.24），短上下文榜变化很小——说明 harness 主要吃超长证据，而非刷短榜虚荣。

## 🔬 最有意思的部分

1. **固定权重也能证明 harness 杠杆**：71.90→77.05 把「脚手架独立贡献」从口号变成配对数字，Skills 跃迁尤其刺眼。
2. **CARE 不是冷酷压缩**：去掉 advantage calibration 会更短但精度平台更低——效率信号必须被锚定，否则 GRPO 会把「少想一点」当成和「做成任务」同权。
3. **共进化 > 只加 harness**：Internal / MCPMark 上 Model+Harness 不够，交替修订才到位；且作者主动降调「自治共进化」叙事。
4. **失败痕迹是一等公民**：保留 failure traces 进诊断与数据/harness 修订，对照 [AEWM](./20260925_AEWM_论文解读_别再猜工具回包_判动作改状态才是Agent世界模型.md) 的污染改写、[SkillPivot](./20260925_SkillPivot_论文解读_失败别整轨反思_先找偏航点再对比后缀改技能.md) 的偏航点——都是把失败当可操作信号。
5. **MCPMark 外溢**：mobile 闭环训出的系统在一般 MCP 工具压力测试上也涨——对 coding-agent 工具面有迁移暗示。

## 🤔 我的判断

定位：**把 Data / Training / Harness 接到同一验证合同上的 AI-for-AI 工程样本**，附带受治理的 model–harness 共进化路径；不是宣称「agent 已能无限自我升级」。亮点三：

1. 任务形式 \(c_t=\mathrm{H}_\eta(\ldots)\) 把 harness 写进策略输入，而不是事后粘贴；
2. CARE 把「先会做、再少花」做成可调度的能力感知奖励，并正面处理 GRPO 效率组放大；
3. 固定 checkpoint 的 harness 消融 + 多轮共进化，把脚手架杠杆与协同杠杆拆开量。

局限：主榜与部分对比模型命名偏产品线/闭源快照，外部复现依赖项目页与后续开源；共进化用的 evolve-set 是开发集而非独立终测；真机比例与人闸门成本未完全摊开；「AI-for-AI」里仍大量依赖 LLM controller / 诊断 agent——治理链本身也要评测。对 Byron：Pi / OpenClaw / coding harness 可直接抄三件事——（1）**serving 冻结权重，harness 指令走版本审查**；（2）把 Skills/Memory/MCP 失败轨迹结构化回流，驱动下一轮数据与脚手架，而不是只开 issue；（3）RL 课表按能力分档，成功饱和后再压工具/思维成本，避免过早压缩。对照 [Paper2Agent](./20260924_Paper2Agent_论文解读_论文别只当PDF_锁死验证过的MCP工具比自由写代码更可靠.md)：工具合同与 verifier 可信，共进化才不会把噪声写进下一版 harness。

一句话收束：agent 系统不是「训完一个 planner」——**Skills、Memory、Harness 要和权重共用执行证据，在审查下共进化；自治神话可以后置，治理路径必须先立住**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
