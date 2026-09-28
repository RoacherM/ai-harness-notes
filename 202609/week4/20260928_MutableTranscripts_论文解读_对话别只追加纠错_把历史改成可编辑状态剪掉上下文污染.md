---
month: 202609
week: 4
date: 2026-09-28
type: 论文解读
slug: MutableTranscripts
---

# 对话别只追加纠错：把历史改成可编辑状态，上下文污染就能剪掉

你有没有过这种体验：跟 coding-agent / ChatGPT 聊到一半才发现约束写错了——年份、国家、预算、技术栈——于是发一串「抱歉我刚才说错了」；线程越来越长，模型却还时不时把**过期设定**捞回来。Gemini / Duck.ai 只让改最后一条；ChatGPT / Claude 允许中间编辑却走 **branch**，旧污染仍躺在原分支。都柏林大学 QxLab 这篇 Mutable Transcripts（arXiv:2609.31354）把诊断钉死：**上下文污染是「不可变 transcript + 只追加纠错」的设计后果**；提案是把对话历史从被动日志改成 **可编辑会话状态**——用自然语言 edit request 整段改写 \(T'=f(T,e)\)，而不是再堆一轮纠正。看完我的感受：这和 [LIMBO](./20260924_LIMBO_论文解读_经验回放别全塞进提示_按任务在线配记忆预算.md) / [JITMem](./20260925_JITMem_论文解读_记忆别在写入时裁剪_读时按任务现场策展同一条轨迹能榨出不同课.md) / [AEWM](./20260925_AEWM_论文解读_别再猜工具回包_判动作改状态才是Agent世界模型.md) 同一条轴——**状态该可修订**，不是永远 append-only；对 harness / MCP 会话层是直接的 UX 与上下文卫生问题。

## 核心摘要

作者把当代 LLM chat 的默认合同概括为：stateless 多轮、唯一「记忆」是每次回传的 JSON 交替消息列表。用户意图会演化（纠错、加约束、砍岔题），系统却只能追加，导致 **context pollution**：过时、矛盾、无关信息继续影响后续生成。Mutable transcripts 保留聊天隐喻，增加「Edit History」模式：自然语言指令作用于整段历史，模型重写后替换展示与后续状态。原型 ReChat（Gemini 2.5 + Google AI Studio，https://github.com/QxLabIreland/ReChat）覆盖三类操作——**回溯纠错、约束注入、上下文剪枝**。对照用户研究 \(n=17\)（多数技术从业者，组内交叉 A/B）：五维 Likert（清洁度、状态清晰、置信、重启意图、易用）上可变条件全面显著优于标准 chat（配对 t 检验均 \(p<0.001\)）；**88.2%** 表示若真实产品有「Edit History」会用。代表性 transcript 分析：平均轮次 **8→4（−50%）**、token **581→238（−59%）**，基线中平均 **71.4%** token 属于被后续纠正 superseded 的陈旧内容，可变条件下 obsolete 清零。定位是概念与可行性研究，非大规模榜单对决。

## 论文信息

- **标题**：Mutable Transcripts: Mitigating Context Pollution through Editable Conversation State
- **作者**：Dan Barry、Andrew Hines
- **机构**：University College Dublin（School of Computer Science）
- **链接**：https://arxiv.org/abs/2609.31354 · 原型 https://github.com/QxLabIreland/ReChat
- **观察时间**：2026-09-28

---

## 🎯 为什么这件事值得写

上下文管理文献多站在系统侧：截断、摘要、RAG 记忆——用户无法显式对齐「现在到底信哪套设定」。Canvas / 笔记本 / agent 框架允许改工件，却往往**丢掉对话隐喻**。Mutable transcripts 卡在中间：还是 chat，但历史可修订。

对 Byron（harness / skills / MCP / agent）：coding-agent 会话里「先错约束再补丁」极常见——Pi Manifest、OpenClaw 多 harness、长工具轨迹都怕污染。和 AEWM「动作改状态」合读：工具世界有状态转移，**对话世界也应有一等状态编辑**；和 JITMem「读时策展」合读：污染是写侧 append 造成的，可变 transcript 是用户驱动的写侧修订。安全与问责（prompt injection 改写历史、审计不可变）作者明确标为开放问题——工程上要配版本/diff/undo，不能裸上生产。

## 🏗️ 机制：双模式 + 整段重写

**默认模式**：与常规 chat 相同，用户输入 append 进 JSON transcript，全量送模型。

**Edit History 模式**：构造「conversation editor」系统提示（附录 A）：把当前 history JSON + 用户自然语言 instruction 交给模型，要求返回**改写后的完整 JSON 数组**（保留 Markdown 列表等结构），UI 刷新替换整段线程。形式化 \(T'=f(T,e)\)。

**代表操作：**

1. **Retroactive correction**：如「把 GDP 问题从 2021 改成 2022，并更新后续所有回答」——单次编辑向下传播。
2. **Constraint injection**：如「全程假设：无云、仅端侧、WCAG AA」——全局回写先验。
3. **Context pruning**：如「删掉气候变化经济影响那段岔题」——选择性切除。

附录 E 还列风格迁移、红acted 分享、导入外部对话等——核心仍是 **transcript = 可编辑状态表示**。作者承认当前实现是全量重生（输出 token ∝ 长度），未来可走 structured diff / 局部更新；全量重写换一致性，用延迟与成本换。

## 🧪 关键证据

**用户研究设计。** \(n=17\)，自愿无报酬，伦理审批；15/17 为研究/软件技术从业者。组内三任务 × 两条件（标准 chat A vs 可变 B），平衡顺序（10 人 A→B，7 人 B→A）。每任务后五题 1–7 Likert。

**主观结果（Figure 3）。** 合并数据上 B 在清洁度、清晰度、置信、易用上更高，**重启意图更低**（更不愿开新会话）；全部 \(p<0.001\)。两组顺序各自内部仍是 B 优——非熟悉度假象。任务前纠错习惯：94.1% 发 follow-up，70.6% 复制改 prompt 重发，仅 35.3% 用「改上一条」（若有），23.5% 会整聊重启——说明不可变历史逼出低效策略。任务后 **88.2%** 愿用 Edit History。

**结构分析（Table 1，代表性三场景各一对）。**

| 场景 | 方法 | 轮次 | Tokens | Obsolete% |
| --- | --- | ---: | ---: | ---: |
| 回溯纠错 | Baseline / Mutable | 8 / 4 | 422 / 169 | 56% / — |
| 约束注入 | Baseline / Mutable | 6 / 4 | 608 / 479 | 58% / — |
| 上下文剪枝 | Baseline / Mutable | 10 / 4 | 712 / 66 | 92% / — |
| **均值** | | **8 / 4** | **581 / 238** | **71% / —** |

Obsolete = 被后续纠正 superseded 的用户+助手轮 token（tiktoken）。作者强调：这是**结构示意**，非全员大规模基准；目标是量级感，不是宣称普适涨点。

**定性反馈。** 正面：更干净、好回读、长线程变短。负面：覆盖可能丢有用旧想法；要 diff / 版本历史 / undo；探索性对话未必需要重写。

## 🔬 最有意思的部分

1. **污染是交互合同问题，不只是长上下文模型能力问题**：Lost-in-the-middle / multi-turn 迷失文献管模型侧；本文管 **用户能否修订状态**。
2. **Branch ≠ Revise**：分支保留探索，但旧污染仍在；修订牺牲可恢复性换当前意图对齐——产品该暴露两种模式，而不是二选一藏起来。
3. **自然语言编辑降低 UX 摩擦**：不必教用户逐条改消息，模式切换即可——符合「少点远程 UI」的偏好信号。
4. **与记忆论文正交**：LIMBO/JITMem 管检索与策展预算；可变 transcript 管 **会话层事实来源**——先剪污染再策展，避免 curator 吃陈旧设定。
5. **安全阴影真实存在**：可编辑历史放大 injection 与问责模糊；作者刻意只做可用性，工程落地必须加 provenance。

## 🤔 我的判断

定位：**交互范式 + 可行性用户研究**（非新压缩算法、非新记忆网络）。亮点三：

1. 把 context pollution 归因到不可变 transcript 设计，命题清晰；
2. 保留 chat 隐喻的同时引入一等修订，原型与 \(n=17\) 证据够「值得跟进」；
3. 结构数字（−50% 轮、−59% token、~71% obsolete）给 harness 团队可抄的卫生指标。

局限：样本量小、任务脚本化、无与 branching 的头对头；全量重写成本与长会话延迟；自然语言编辑歧义与「误删有用上下文」；安全/审计未解；多数被试偏技术圈。对 Byron：在 Pi / OpenClaw / 多 harness 会话里，**值得加「Edit History / 修订会话状态」能力**——至少支持约束注入与剪枝，并强制写版本链（before/after diff、可 undo）；评测指标可加「obsolete token 占比」与「重启率」。别把系统侧摘要当成用户意图修订的替代——摘要是压缩，**修订是对齐**。和 ExactlyOnce / 工具合同合读：工具侧要恰好一次，对话侧也要 **当前有效约束恰好一份**。

一句话收束：意图会变，历史却不该只能越堆越脏——**把 transcript 当成可编辑状态，污染才能剪掉，而不是靠下一句道歉去盖住**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
