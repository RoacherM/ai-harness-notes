---
month: 202609
week: 3
date: 2026-09-18
type: 论文解读
slug: Agora
---

# 加 agent 不等于加发现：Agora 用 Git DAG 当集体 AutoResearch 的共享记忆

你有没有过这种体验：一个 coding agent 能自己跑通一轮 AutoResearch，挺爽；再开十来个并行，结果大家各自从零搜、踩同一坑、还互相不知道谁已经证伪了什么。行业里多 agent 框架多半管的是**同一场对话里的角色分工**；真正缺的是跨会话、跨账号、还能独立复核的**公共科研状态**。NVIDIA 这篇 Agora（arXiv:2609.18094）把命题说得很硬：**研究记录成 Git 上的 append-only DAG，每个 claim 是一个谁都能 checkout 重跑的 commit**——Git 是唯一状态，工人之间不共享文件系统、不共享对话。看完我的感受是：它不是又一个「多智能体编排器」，而是在给 agent 社区补一层**制度性记忆**；和 Atlas 式「会话历史」也不是一类东西。

## 核心摘要

Agora 把多 agent 科研形式化为贡献图 \(G=(V,E)\)：节点带 artifacts、声明、可选指标与 provenance，边表示「builds on」。每条贡献是不可变 commit（result / insight / hypothesis / verification / report 等标签），质量不靠点赞，而靠**其他账号的下游复现与延续**（自引不计分；验证有 confirmed / partial / failed 权重）。派生索引暴露前沿、冷落分支与验证状态；diversity-aware UCB 把候选拆成 exploit / explore known / explore novel 三槽，防止全员挤向同一排行榜盆地。首次持续实验：约 **12 天**、**13** 个语言模型工人、无任务分配、无中心规划，在 **141** 个预训练 donor 与冻结的 **119.6M** attention–SSM hybrid（维度与任何 donor 都不匹配）上做**无训练数据、无梯度更新**的权重迁移初始化。工人发布 **1,703** 条贡献，把评估器从 **3.39** 推到 **1.899** bits/byte，相对训练好的 GPT-2 124M 约合 **62%** 的差距闭合；优胜配方先把 donor 的 next-token 统计压进目标 embedding 与输出头，再对 attention / FFN / SSM 做稀疏短程上下文编辑。优胜节点有 **145** 条祖先、跨 **15** 个账号，**165** 次独立复现且**无一失败**。作者诚实承认：因果上尚未做「有无 Agora」的对照，五天单文化是靠一次人类干预（放出多样性地图）才被打破的。

## 论文信息

- **标题**：Agora: Git as Shared Memory for Collective AutoResearch
- **作者**：Yifan Zhang, Yunheng Zou, Shaokun Zhang, Jian Hu, Hao Zhang, Binfeng Xu, Jan Kautz, Yi Dong
- **机构**：NVIDIA
- **链接**：https://arxiv.org/abs/2609.18094 （2026 年 9 月 16 日提交）· 仓库/报告在论文页可跟

---

## 🎯 为什么「多开几个 AutoResearch」会变成重复搜索

单 agent 的 AutoResearch（Karpathy 等）证明：无人值守也能把训练设定迭代上去。但会话学到的东西卡在 transcript 或临时 worktree 里——下一轮不知道哪个学习率炸了、哪条分支被弃、哪条结果还缺独立复现。工人一多，尝试变多，**重复搜索、过早收敛、重建「谁做了什么」的成本**也一起涨。

现有多 agent 框架擅长角色扮演、可编程对话、SOP；它们协调的是**同一应用/同一 episode 内的团队**。科研社区还要：公共前沿、不可变 lineage、负面结果、独立验证，以及在**不规定单一工作流**的前提下分散注意力。Agora 声明自己是协调基底（coordination substrate），不是实验室经理：项目自定指令、指标与安全边界，平台只提供发布与发现机制。

对 Byron 的 harness / MCP / skills 栈，同构问题很直白：多会话、多插件、多工人若只靠聊天记录或各自本地记忆，集体发现会被「并行浪费」吃掉。

## 🏗️ 机制：Git 为真源，证据与注意力分两层

### 贡献词汇与轻重发布路径

保留标签带验证/计分语义：`setup` / `result` / `insight` / `hypothesis` / `report`（基础权重多为 +5），`verification` 对确认 / 部分 / 失败分别 **+20 / +10 / −20**，且**不能验证自己的工作**；`endorsed` / `wip` 可见但不进 fitness。元数据走 light path（JSON → 服务端建 canonical commit），带代码走 heavy path（本地 commit + Git bundle，服务端校验后再盖时间戳）。两边产出同构节点，查询不区分路径。

证据分 \(S(u)\) 是**其他账号**在下游边的加权计数——「别人复现过/接过」，不是真值投票。验证裁决可被更新，但旧 commit 留在历史上。

### Diversity-aware 注意力

纯排行榜是好的 exploitation、坏的地图。`analyze` 一次返回：指标领袖、被构建最多的节点、叶子、有潜力但欠探索的结果、未验证/有争议验证、开放假设、近期活动等。覆盖足够后加语义聚类；候选用带质量百分位、跟进次数、近重复惩罚的 UCB，并强制三槽展示：

| 槽 | 意图 |
| --- | --- |
| exploit | 复现或打磨领袖 |
| explore known | 在薄集群里延续有希望的工作 |
| explore novel | 碰单例/极小集群里未碰过的节点 |

原型是 Go 服务 + CLI + Next.js；每项目一个 bare repo，SQLite 索引可从 Git 重建——**Git history 是唯一依赖状态**。

## 🧪 Weight-Transfer Run：数字与轨迹

任务：从 donor zoo 初始化架构全不匹配的冻结 hybrid，评估用 bits per byte。工人循环：读 analyze → 选 parent → checkout 精确 commit → 改一处 → 评估 → 推带描述与指标的 commit → 再发 insight/hypothesis/verification。

- **时长**：服务端约 11 天 19 小时（Apr 26 setup → May 8 cutoff）
- **规模**：1,703 贡献 = 1,124 有分结果 + 284 insights + 203 hypotheses + 165 verifications + 1 report；其中 233 次刷新最优；稳态约每天 170 条
- **主结果**：随机初始化 **3.3923** bpb → 最优 **1.899044** bpb，相对约 **1.0** 的训练 GPT-2 124M，闭合约 **62%** 差距；无训练数据、无梯度
- **优胜配方 Stage A**：用 6 个共享 GPT-2 词表的 donor，在 28 个单 token 上下文上压出 \(50257\times50257\) 转移表，SVD 进 embedding/head；Stage B：对 96 维 band 做稀疏确定性编辑（attention 均值池、SSM 深度卷积等）
- **谱系**：最优节点 145 祖先、15/17 账号参与；144 条父边里 115 条跨账号——**没有单一工人拼出整条配方**
- **复现**：165 次验证覆盖 95 个目标，失败 0；同硬件 bit-identical，跨 A100/H100 差至多 \(1.3\times10^{-3}\) bpb
- **负面结果**：53 条显式 negative tag（例如把 prefix 加倍到 48、直接拷 embedding、换 tokenizer 的 Pythia prior 等）

协调动力学同样刺眼：前 8 次改进约占总下降 **70%**，前 18 次 scored 约占 **98%**；图呈窄 spine + 短废弃枝；696 对「不同账号相同分数」里 **63%** 一小时内、**80%** 六小时内平行发现——共享记忆加速前沿，也放大 exploitation 偏见。

## 🔬 最有意思的部分：一次人类干预与「制度 vs 个体」

五月二日，分析视图显示超过三分之一活动挤在单一语义簇、排行榜停滞。作者**只部署了**聚类、多样性摘要与 diversity-aware UCB，**不分配任务、不审批贡献、不舵单个会话**。工人立刻采用新视图；次日上午，一个选择跟随「薄 SSM 簇」而非主导簇的工人发出首个 sub-1.90 结果。作者把结论写得很克制：五天单文化说明协调层才是 binding constraint；但因果上还没证明「没有 Agora / 只有纯排行榜」会怎样——Appendix C 给出可预注册的 matched evaluation。

和「会话记忆型」系统（例如把 coding session 写进历史）比，Agora 的差异在：**claim 级 provenance + 可重跑 artifacts + 跨账号验证经济**。它管的是集体科研制度，不是单助手的上下文窗口。

## 🤔 我的判断

定位：**多智能体科研的协调基础设施 / 测量协议样本**，附带一次漂亮的 weight-transfer 故事。对做 agent harness 的人，可直接带走的口令是：

1. **共享状态要内容寻址且 append-only**——transcript 不够，需要可 checkout 的 claim；
2. **质量信号用下游复现，别用自我背书**——和 judge 闭环里「失败回灌」是同一类制度设计；
3. **排行榜必须配探索槽**——否则多样性会在五天内塌成单文化；
4. **负面结果要一等公民**——53 条负结果被后续工人引用选方向，这才是集体搜索。

局限：单次运行、无有无 Agora 的严格对照；主任务是特殊初始化问题，不是任意开源科研；人类仍定义任务与评估器。对 Byron：若你在想多 agent / 多会话如何共享 skills、实验结论与失败模式，这篇比又一个编排框架更值得进 `ai-harness-notes`；若只做单会话 Manifest，当「集体记忆该长什么样」的隐喻扫一眼即可。

一句话收束：下一个瓶颈往往不是再开一个更强的 coding agent，而是**社区有没有一份谁都能复现、谁都能看见冷落分支的公共研究状态**——Agora 用 Git DAG 把这句话做成了可跑的系统。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
