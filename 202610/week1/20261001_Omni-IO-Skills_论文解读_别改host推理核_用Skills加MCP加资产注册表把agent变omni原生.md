---
month: 202610
week: 1
date: 2026-10-01
type: 论文解读
slug: Omni-IO-Skills
---

# 别改 host 推理核：用 Skills + MCP + 资产注册表，把已有 agent 变成 omni-原生

你有没有过这种体验：Claude Code / Codex 规划与改仓很强，可一旦任务变成「看完这段视频 → 抽步骤 → 配 BGM → 出封面与课件」，能力立刻碎成一堆外挂模型——谁管依赖、中间文件怎么跨轮复用、换供应商要不要重写流程，全靠人手焊。NUS / Oxford 这篇 **Omni-IO Skills**（arXiv:2609.31847）给出一条清晰路线：**不动 host 的推理核，用可插拔 Agent Harness（分层 Skills + MCP 工具面 + 依赖感知编排 + 持久 Asset Registry）把通用 agent 变成 omni-原生**。看完我的感受：这几乎是「skills / MCP / harness」三件套的教科书拼装——用 Declare Execution Graph 把多资产工作流变成可并发、可追溯的执行面，而不是再训一个更大的 Omni 基座。

## 核心摘要

Omni-IO Skills 是 plug-and-play **Omni-modal Agent Harness**：在 host agent 与异构多模态后端之间插入四层——**Skill Entry / MCP Tool Service / Provider & Config / Asset Registry**。Skills 分 Atomic / Expert / Scenario 三级（19+2+6=**27** Skills，覆盖 **38** 类代表任务、**七**种制品模态：文本/图/音/视/文档/3D/代码，以及理解·生成·推理·检索四族能力）。多资产流程写成 **Declare Execution Graph（DEG）**：显式控制/数据依赖、校验无环后按 Wave 并发调度，失败取消子孙、独立分支继续；成功输出写入持久 **Asset Registry**（asset_id 与物理路径解耦，支持跨轮 `asset_ref` 修订）。后端可替换，Skill 只绑语义能力不绑厂商。评测 **UniM-90**：GPT-5.6 Sol / Claude Sonnet 5 的输入支持率从 **40% / 38.9% → 100%**；相对 Semantic–Quality Coupled Score 从 **27.0 / 27.8 → 74.9 / 77.8**（约 +48 / +50 pp）；Strict Structure Score 达 **100 / 99.78**。结论：harness 级能力组合是通向可演化 Omni 系统的可行路径。仓库：any2any-mllm/Omni-IO-Skill。

## 论文信息

- **标题**：Omni-IO Skills: Harnessing Your Agent Omni-Native
- **作者 / 机构**：Yanlin Li, Mingyang Hao, Shengqiong Wu, Hao Fei（通讯）, Mong-Li Lee, Wynne Hsu（NUS · Oxford）
- **仓库**：https://github.com/any2any-mllm/Omni-IO-Skill
- **链接**：https://arxiv.org/abs/2609.31847 · https://arxiv.org/pdf/2609.31847
- **观察时间**：2026-10-01（HF Daily Papers / AK 9-30 精选前列；skills · harness · MCP）

---

## 🎯 为什么这件事值得写

Omni 基座把更多模态塞进同一权重，成本跟训练周期绑死；「主 agent 调专家」又常停在问答/取证，缺**多交付物依赖、中间资产交接、跨轮修订**。Agent Skills 已证明可复用程序知识，MCP 标准化了工具暴露——缺的是把二者焊成生产工作流的 **harness**：选能力、声明依赖、搬资产、隔离失败、跨轮找回。Omni-IO 正好落在 Byron 的 skills / harness / MCP 交叉点：host 仍是 Claude/Codex 一类通用 agent，能力扩张走 Skills 与 provider 配置，而不是改推理核。

## 🏗️ 机制：四层 + DEG + 资产生命周期

**Skill 合同。** \(s=\langle c_s,I_s,P_s,O_s,H_s\rangle\)：适用条件、语义输入输出、过程、与下层 Skills 关系。Scenario 定应用边界与交付物组合 → Expert 封一条专业生产线 → Atomic 是可调度原语（外部走 MCP，代码/Markdown 等可由 host 原生执行）。

**DEG。** 节点含 id/type/prompt/params/depends_on；边同时是控制依赖与数据依赖。校验引用可解析且无环后，就绪集组成 Wave 并发；模态无关——图生视频等图，图与音频可并行。历史资产以 `asset_source` 节点直接视为完成，不重复生成。

**Asset Registry。** 统一记录 type/subtype/path/description/params/turn_id/source_asset_id；追加写 + 文件锁，修订产生新记录而非原地覆盖。下游用 asset_id 取路径，prompt 不必内嵌绝对路径。

**Provider 解耦。** Skill 说「图像生成」，绑定层才选模型/凭证/默认参数/fallback——换供应商不改程序知识。

## 🧪 关键证据

UniM-90（UniM 固定 90 例，覆盖七模态及交错组合；与任一 agent 原生能力无关地选取）：

| Host | \(\tau\) 支持率 | 相对 SQCS | StS |
| --- | --- | --- | --- |
| GPT-5.6 Sol 基线 | 40% | 26.99 | 19.14 |
| + Omni-IO | **100%** | **74.94** | **100** |
| Claude Sonnet 5 基线 | 38.89% | 27.82 | 20.30 |
| + Omni-IO | **100%** | **77.78** | **99.78** |

绝对 SQCS 也上涨（67.5→74.9 / 71.5→77.8），说明不止「拓宽能接的输入」，在已支持子集上质量也升。案例：教育分享（视频+音频理解 → 多图教程）、产品推广（图理解 → 海报/视频 Expert → 落地页代码），两 host 均能交齐交付物并保持一致性。

泼冷水：评测是受控 90 例子集而非全 UniM；跨 agent 分数不宜当模型排名；结构分接近满分可能部分来自注册表与工具合同约束，语义质量仍有 20+ 分空间；生产可靠性（计费、审核、长视频成本）文中未做压力测试。

## 🔬 最有意思的部分

1. **「能力长在 harness，不长在权重」**——与 Raven / Meta-Harness 同族叙事，但是多模态生产面的落地样本。
2. **Skills 分级不是营销分层**：Atomic 可单步调用，Expert 含质检与局部返工，Scenario 管多交付物选型——这比「一个大 skill.md」更可演化。
3. **DEG Wave 调度**把并行从「模型自己想」上收到声明式图，失败语义清晰（取消子孙、保留已完成资产）。
4. **MCP 被放在「语义任务 → 可执行工具」的边界层**，而不是让 Skill 正文写死 HTTP——换后端只动配置。
5. **Asset Registry 是跨轮记忆的窄而硬形态**：不做泛化向量库，只保证制品可指称、可溯源、可修订——对工程更可运维。

## 🤔 我的判断

定位：**skills + MCP + 依赖图编排的 omni harness 工程样本**（系统论文，附双 host 对照）。亮点三：

1. 明确「保留 host 推理核、扩张走 harness」的产品边界；
2. DEG + Asset Registry 把多资产工作流做成可调度状态机；
3. UniM-90 上支持率与结构分的跃迁，证明缺口主要在编排层而非「再换一个更强聊天模型」。

局限：偏生成/内容生产场景，对 SWE / 长程工具 agent 的迁移要另证；27 Skills 覆盖是策展清单不是开放世界；缺少与「纯提示拼工具」或「单一 Omni 模型」的成本–质量全对比。对 Byron：**(1)** MMP Manifest 可直接对标四层：Skills 条目 / MCP 桥 / provider 绑定 / 制品账本；**(2)** 写 skill 时强制「语义 I/O + 可扩展下层」，禁止在正文写死厂商 API；**(3)** 跨轮复用优先 **asset_id 合同**，别只靠聊天摘要记文件名；**(4)** 接 [EvoSkill-GUI](../202609/week3/20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) / [Paper2Agent](../202609/week4/20260924_Paper2Agent_论文解读_论文别只当PDF_锁死验证过的MCP工具比自由写代码更可靠.md)：Omni-IO 补的是**多模态制品工作流的 harness 骨架**。

一句话收束：**要 omni，不必先重训 host——用分层 Skills、MCP 与资产注册表，把已有 agent 编排成可演化的多模态生产线。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
