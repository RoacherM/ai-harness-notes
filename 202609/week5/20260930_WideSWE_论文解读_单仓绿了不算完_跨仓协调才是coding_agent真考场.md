---
month: 202609
week: 5
date: 2026-09-30
type: 论文解读
slug: WideSWE
---

# 单仓绿了不算完：跨仓协调，才是 coding agent 的真考场

你有没有过这种体验：SWE-bench / DeepSWE 上模型「修完一个 issue」看起来很能打，一换到真实生态——Sentry 要同时改 Go / Python / Ruby SDK，或 Laravel AI 修流式还得动 MCP 集成——agent 在一个仓把测试跑绿，另两个仓纹丝不动，整单仍算失败。浙大 ACES + 清华这篇 **WideSWE**（arXiv:2609.33382）把问题钉死：**任务成功 = 所有目标仓都过 F2P+P2P**，不是「至少一个仓有进展」。看完我的感受：这正好接在 Byron 的 **coding-agent 评测 / harness 工作区设计** 线上——单仓榜已经不够诊断「会不会跨边界交付」；轨迹里三种失败（范围认不全 / 认了不交付 / 改了仍不满足）比总分更能指导脚手架。

## 核心摘要

WideSWE 从 103 个软件生态、约 172 万条跨仓引用 PR 中筛出 **120 个真实跨仓任务**（60 bugfix + 60 feature，覆盖 41 生态、253 目标仓），要求 agent 在含上下文仓的历史工作区里，对**同一功能/修复**完成至少两个目标仓的协同改动；隐藏测试经「放宽实现细节 / 去掉未请求功能」的人工对齐，避免因私有 helper 名或多余预设拒掉正确实现。七组配置（Codex CLI / Claude Code × GPT-5.6-sol、Opus 5、Gemini 3.8 Flash、DeepSeek V4 Pro、Qwen 3.8 Max、GLM 5.3）整案成功率仅 **10.83%–42.50%**（最佳 Codex+GPT-5.6-sol），但仓级成功可达约 63%，未解案中大量「部分仓 F2P 全过」。轨迹三分：不完全 scope、认了不交付、改完仍挂测。联合执行 vs 分仓独立执行（同 prompt、89 题）：联合 40.45% vs 独立 35.96%，独立更能捡回「漏改的仓」，却丢掉跨仓对照带来的实现/验收线索；独立 API 请求约 3.1×。仓库：https://github.com/ZJU-ACES-ISE/WideSWE。

## 论文信息

- **标题**：WideSWE: Can Coding Agents Coordinate Changes Across Repositories?
- **作者**：Baoyi Wang*、Xingliang Wang*、Jinyang Wu、Keming Wu、Chen Zhi†、Jianwei Yin（* 共一；† 通讯）
- **机构**：浙江大学；清华大学
- **链接**：https://arxiv.org/abs/2609.33382 · https://arxiv.org/pdf/2609.33382 · https://github.com/ZJU-ACES-ISE/WideSWE
- **观察时间**：2026-09-30（HF Daily Papers / AK digest 覆盖的 Sep 29 榜）

---

## 🎯 为什么这件事值得写

SWE-bench 系、DeepSWE、ProgramBench 把评测从「写函数」推到「在真实仓里干活」，但仍默认**一个目标代码库**。工业里跨仓引用极常见：作者统计 1,729,171 条 PR 中 109,233 条显式指向同生态另一仓。BeyondSWE 的 CrossRepo 侧重「用外仓信息修目标仓」；WideSWE 反过来：**多个目标仓都要交付且联合验收**。对 harness：这逼出工作区形态（多仓挂载、上下文仓上限 20）、成功定义（乘积式 Success）和失败切片（哪一类断在协调链上）。

## 🏗️ 机制：任务形式化、造榜与评分

**任务。** 请求 \(p_c\) + 历史工作区 \(\mathcal{W}_c\)（仓集 \(\mathcal{R}_c\)），目标子集 \(\mathcal{T}_c\subseteq\mathcal{R}_c\)，\(|\mathcal{T}_c|\ge 2\)。Agent 产出补丁集 \(\Delta_c\)，仅目标仓计入得分；上下文仓可读写但不决定成败。

**造榜。** 顶星组织筛 103 生态 → 链联合并 PR → 人工确认「单一功能/修复且多仓实质改动」→ 可执行校验（每目标仓至少一条 F2P）→ 平衡 60/60。Prompt 合并原始 issue/PR 表述，刻意不加逐步实现说明书。隐藏测试从参考补丁提取后按两条规则修订：放宽实现专有约束；删除未请求功能检查——并复验 Base 仍失败、Gold 仍通过。

**评分。** 目标仓 \(r\) 上全部 F2P∪P2P 通过才 \(y_{c,r}=1\)；\(\operatorname{Success}(c)=\prod_r y_{c,r}\)。另报仓级成功、F2P-complete、P2P-preserved、人均 API 请求。

**协调模式。** 120 题中约 80% 为 producer–consumer 依赖，20% 为并行传播（多语言 SDK 等同行为）。

## 🧪 关键证据

**主表（Overall）。** Codex+GPT-5.6-sol：Case 42.50 / Repo 63.64 / F2P 65.61 / P2P 90.95 / API 86.7。Claude Code 同模型落到 32.50 Case、API 升到 134.8——**同模型换脚手架成本升、分降**。Gemini+CC 仅 10.83 Case，失败里 52.34% 属「认了不交付」（大量停在调查/规划）。Qwen+CC bugfix 强（48.33）但 feature 弱（26.67），多语言家族 feature 更掉——**任务类型 × 语言多样性**不能用单一难度轴概括。

**未解案结构。** 除 Gemini 外，多数未解案仍有「至少一个仓 F2P 全过」；纯 P2P 拖死少见。Codex 失败三分：Scope 37.68 / Delivery 2.90 / Post-edit 59.42。典型：三仓只改 Go；Godot 诊断在 call site 却只改 Native；两仓都改但表示跨仓契约（Sentry/Symbolicator 用异常名字串代替结构化异常）本地测过、正式测挂。

**Joint vs Independent（89 题同 prompt）。** 联合整体更好，但 bugfix 独立升、feature 联合升；联合失败且**未改**的仓独立恢复约 60%，已改错的仓独立只恢复约 9%——**分仓主要捡漏，难救坏实现**。联合独有成功常因对照邻仓实现/行为（Ansible 查表字段、Symfony 双端一致性）。

**泼冷水。** 目标仓多为 2–3 个、Linux 环境；部分 prompt 需仓特异改写故未进 Joint 对照；隐藏测试修订覆盖 88/120 题——评测仍依赖作者对齐质量；不声称覆盖「十仓级」协调。

## 🔬 最有意思的部分

1. **仓级进度 ≠ 任务完成。** 83% 任务至少一仓有进展 vs 42.5% 整案成功——日报若只报「修了某个仓」会系统性乐观。
2. **脚手架不是中性容器。** 同 GPT-5.6-sol，Codex vs Claude Code 差 10 pp Case；轨迹对比显示 scope 复查习惯差异（Propshaft/Rails）。
3. **Gemini 的 Delivery 黑洞。** 计划写进轨迹却数百次读搜不落补丁——评测要单独切「recognized-not-delivered」，别全算能力不够。
4. **共享错误假设可通过本地测。** 两端一致地错 + 自写测共谋——提醒：跨仓契约需要**跨边界的正式检查**，不能只信各仓本地绿。
5. **独立执行不是免费放大。** 3× API 不换整体 Case——预算应优先给「联合工作区里的对照与验收」，而不是机械拆 N 次单仓跑满。

## 🤔 我的判断

定位：**跨仓 coding-agent 的任务形式化 + 可执行基准 + 失败机制研究**（测量基础设施，附脚手架/模型对照）。亮点三：

1. 成功定义强制「联合交付」，直接打穿单仓榜的盲区；
2. 测试对齐规则可复用到任何从 PR 挖任务的流水线；
3. Joint/Independent 把「漏改」和「缺对照」拆开，对 harness 编排有可操作含义。

局限：仓数与平台有界；Industrial org 私有协调模式未覆盖；LLM 轨迹标注虽有 κ=0.86 仍贵。对 Byron：若你在搭 MMP / coding harness——**(1)** 工作区默认支持多仓挂载与「目标仓 vs 上下文仓」权限；(2) 门禁用 \(\prod\) 式跨仓 pass，并切片 Scope/Delivery/Post-edit；(3) 对 feature 类任务优先保留联合上下文，对「明显漏仓」可再开独立补跑，而不是一律拆仓；(4) 契约面加跨仓集成测，防本地绿共谋。接本周 [Gagar](./20260930_Gagar_论文解读_测试过了还不够_组内agentic裁判才把信用拨给可合并补丁.md)（过测内质量）与 [TraceDance](./20260930_TraceDance_论文解读_部署痕迹变成行为基准_决策点续写才测过程.md)（过程行为）：WideSWE 补的是**空间边界上的协调完备性**。

一句话收束：**一个仓的绿灯只是局部进度条——WideSWE 要的是整条生态链路同时亮灯。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
