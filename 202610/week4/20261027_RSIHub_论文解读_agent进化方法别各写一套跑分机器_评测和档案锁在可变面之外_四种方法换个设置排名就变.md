---
month: 202610
week: 4
date: 2026-10-27
type: 论文解读
slug: RSIHub
---

# agent 进化方法别各写一套跑分机器：评测和档案锁在可变面之外，四种方法换个设置排名就变

你有没有过这种体验：读完一篇 harness 自进化论文，想在自己的 agent 上复现，结果发现它自带一整套 runner、一套工作区管理、一套评测脚本、一套 lineage 记录，和另一篇论文的那套完全不兼容。想比较 GEPA 和 AHE 谁更适合你的场景，先得花两周把两套机器对齐。更危险的是，有些方法允许 agent 改「几乎所有东西」，你很难确认它涨的分是真的变强了，还是顺手改了评测、或者改了记录。

这类工作现在一般叫 **agent evolution** 或者 harness 层的 **RSI（recursive self-improvement）**：用执行反馈去持续修改提示、skill、工具和编排。核心问题是：**能不能把「进化策略」和「执行、评测、恢复、溯源」这套反复重建的机器拆开，让方法之间在同一条起跑线上比较，并且让评测永远不在被进化对象的手够得着的地方？**

伯克利、中科大、抖音集团、上交、同济、北京通用人工智能研究院和 Simple Agent Lab 的 **RSIHub: A Composable Framework for Agent-Evolution Methods**（arXiv:2610.32253）给出了一个工程答案。看完我的感受：这是一篇「框架论文」，但它真正有意思的是两个副产品，一是 420 次故障注入和越界攻击全部守住，二是四种主流方法放在同一套机器上，**没有一种方法在所有设置里都赢。**

## 核心摘要

RSIHub 把一次进化迭代固定成七个阶段：Select、Rollout、Analyze、Mutate、Guardrail、Evaluate、Absorb。研究者写一个 recipe（YAML），给每个阶段槽位绑定一个 operator 实现，并声明 `surface`（候选者可以改哪些文件）和 `mutate`（编辑 agent 怎么跑、能看到哪些上下文）。框架统一负责工作区、快照、评测执行、产物留存、中断恢复和 lineage；**评测和档案始终在声明的可变面之外，由框架持有**，operator 只能提出注释，由框架过滤后写入，不能改认证过的评测字段。作者用这套接口实现了 AHE、Hyperagents、A-Evolve、GEPA 四种方法的改编版，每个 recipe 47 到 61 行，方法专属 operator 代码 216 到 1,087 行，**框架源码零改动**。在 MiniSWE 和 Codex 两个种子 agent、Terminal Bench 2 和 τ³ Banking 两个基准上（目标模型 GPT-5.4 high，变异 agent 是 GPT-5.4 xhigh 驱动的 Codex，最多 10 代），**最大整体 pass@1 提升 13.5 个点（MiniSWE 在 Terminal Bench 2 上从 56.2 到 69.7），但方法排名随设置变化，训练集上的增益不总能迁移到 gate 和 sealed 留出集。** 七类共 420 次故障注入与越界试验全部达到要求结果，无一次接纳被污染的状态；框架自身开销只占每代端到端时间的约 0.02%。它的价值定位是：给 harness 进化研究提供一套公共协议和可审计的候选历史。

## 论文信息

- **标题**：RSIHub: A Composable Framework for Agent-Evolution Methods
- **作者**：Zimu Wang、Xiaobo Wang、Siyu Ye、Jiaqi Li（共同一作）、Silin Chen、Yuling Shi、Ruobing Wang、Jingjing Zhang、Chaofan Wang、Kai Wu、Mengnan Qi、Kai Cai、Xiaodong Gu、Zilong Zheng
- **机构**：加州大学伯克利分校、中国科学技术大学、抖音集团、上海交通大学、同济大学、北京通用人工智能研究院、Simple Agent Lab
- **链接**：https://arxiv.org/abs/2610.32253 （v1，2026-10-23）
- **代码**：https://github.com/simple-agent-lab/RSIHub （Apache-2.0）

---

## 🎯 为什么这件事值得写

这两周归档的 harness 进化论文已经不少：RiseHarness 对照成败轨迹改 harness，HarnessAdapt-RL 让工具和 hook 跟着策略一起进化，CoEvo-RSI 讨论脚手架进化后模型跟不上。几乎每一篇都自己造了一套实验机器，而它们共享的生命周期其实高度一致：选一个候选扩展、收集它的行为证据、提出并评测修改、把结果留给后续搜索。

作者指出两个代价。第一，实现一个新方法往往要再造一个端到端系统；比较方法又要先对齐 runner、配置和评测接口。第二，也是更要命的：**在可变范围很广的进化系统里，方法可以改评测、可以改记录**，前者改变测量条件，后者改写证据，任何一种都会让「涨分」变得含糊。相关工作部分作者还引用了奖励模型过优化和 reward tampering 的研究，说明这不是杞人忧天。

所以 RSIHub 的设计原则只有四条：方法用可替换零件拼装；显式声明方法能改什么、能看什么；其余流水线只实现一次；**评分永远不在方法够得着的地方。**

## 🏗️ 机制：一次候选事务

框架底层用 Harbor 做容器执行、轨迹采集和逐任务验证，RSIHub 在其上管理候选 lineage、train/gate/sealed 划分、可变面强制和认证评测记录。

**五个必选 operator 加四个可选 operator：**

| 阶段 | 作用 | 框架做什么 |
| --- | --- | --- |
| Select | 从档案里按搜索策略选父代 | 校验身份，fork 记录在案的精确 commit |
| Rollout | 在训练任务上跑父代，返回证据 | 只用 train 划分；也可重放父代的已记录评测 |
| Analyze* | 把轨迹蒸馏成有界的变异证据 | 打包给 Mutate |
| Mutate | 在可变面内提出修改，通常是多轮 agent | 报告改动路径、说明和用量 |
| Guardrail | 对比真实 tree diff 和声明的可变面 | Validate* 做方法专属预检，Novelty* 丢弃近重复 |
| Evaluate | 提交审核后的树，在干净 checkout 上跑 gate 任务 | 认证结果并写入档案 |
| Absorb | Gate 决定是否成为父代，Record 提注释，Reflect* 提炼经验 | 都不能改认证字段 |

每个 operator 在独立进程里运行，输出先过 schema 校验再被使用。每次迭代前框架会校验并快照当前 recipe。

**认证评测事件**把被评候选、父代、recipe 快照和评测器指纹、任务集指纹、trial、分数、成本、产物引用、lineage 绑在一起；严格模式下还有哈希引用的 contract，记录评测器配置和数据集内容摘要、划分与重复设置、模型身份、重试策略、评测运行时摘要和框架版本，`contract_id` 是整个载荷的规范摘要。父代可选的条件很苛刻：评测完整、可选、没有待定的 Gate 决定、有认证回执、固定比较身份一致、分数有限、generation tag 与 commit 一致。sealed 评测是单独的、不可被选为父代的 anchor，默认不进入变异反馈包。

**两种驱动模式。** recipe 驱动时，研究者写的 recipe 在各代之间固定；agent 驱动时，外层 agent 可以根据允许的反馈在迭代之间改 operator 绑定和策略设置，但目标、评测器、运行时、数据划分和允许的可变面都不变，sealed 任务不进入反馈。论文实验只评了固定 recipe。

**operator 自己也能进可变面。** 像 Hyperagents 那样允许改进化策略本身时，框架对每个阶段都从提供给该阶段的 checkout 加载实现：已经跑过的阶段改了要下一代才生效，Validate 和 Novelty 改了可能影响当前提案，Gate 和 Record 从已提交的子代加载。

附录给了一个 GEPA 风格 recipe 的样子，`surface.include: [target/**]`，`mutate.config.expose_gate_data: false`，`validate.operator: minibatch_improvement`，`evaluator.split` 是 train 0.562 / gate 0.213 / sealed 0.225，基本就是「harness 即仓库、评测闭环写成配置」的形态。

## 🧪 关键证据

**设置：** Terminal Bench 2 共 89 题，切成 50 train / 19 gate / 20 sealed；τ³ Banking 共 97 题，切成 50 / 20 / 27。每题一次 trial，报告 pass@1。表里的划分列是该划分对全量 pass@1 的贡献，三列相加等于 Overall。

**Terminal Bench 2：**

| 种子 | 方法 | Train | Gate | Sealed | Overall |
| --- | --- | --- | --- | --- | --- |
| MiniSWE | Seed | 32.6 | 12.4 | 11.2 | 56.2 |
| MiniSWE | AHE | 37.1 | 12.4 | 15.7 | 65.2（+9.0） |
| MiniSWE | Hyperagents | 36.0 | **16.9** | **16.9** | **69.7（+13.5）** |
| MiniSWE | A-Evolve | **40.5** | 12.4 | **16.9** | **69.7（+13.5）** |
| MiniSWE | GEPA | 36.0 | 12.4 | 15.7 | 64.0（+7.8） |
| Codex | Seed | 39.3 | 15.7 | 14.6 | 69.7 |
| Codex | AHE | 42.7 | 12.4 | 16.9 | **71.9（+2.2）** |
| Codex | Hyperagents | 41.6 | 13.5 | 15.7 | 70.8（+1.1） |
| Codex | A-Evolve | 40.5 | 15.7 | 14.6 | 70.8（+1.1） |
| Codex | GEPA | 39.3 | 15.7 | 14.6 | 69.7（+0.0） |

**τ³ Banking：**

| 种子 | 方法 | Train | Gate | Sealed | Overall |
| --- | --- | --- | --- | --- | --- |
| MiniSWE | Seed | 12.4 | 6.2 | 9.3 | 27.8 |
| MiniSWE | AHE | 16.5 | 7.2 | 5.2 | 28.9（+1.1） |
| MiniSWE | Hyperagents | 17.5 | 8.2 | 7.2 | **33.0（+5.2）** |
| MiniSWE | A-Evolve | 12.4 | 6.2 | 9.3 | 27.8（+0.0） |
| MiniSWE | GEPA | 11.3 | 11.3 | 7.2 | 29.9（+2.1） |
| Codex | Seed | 16.5 | 5.2 | 3.1 | 24.7 |
| Codex | AHE | 17.5 | 8.2 | 8.2 | **34.0（+9.3）** |
| Codex | Hyperagents | 15.5 | 6.2 | 7.2 | 28.9（+4.2） |
| Codex | A-Evolve | 15.5 | 7.2 | 6.2 | 28.9（+4.2） |
| Codex | GEPA | 16.5 | 9.3 | 7.2 | 33.0（+8.3） |

几条读法：

- 弱种子（MiniSWE）在 Terminal Bench 2 上四种方法都涨 7.8 到 13.5 点；强种子（Codex）最多只涨 2.2 点，GEPA 原地不动。
- Codex 在 Terminal Bench 2 上，AHE 和 Hyperagents 训练和 sealed 都涨，但 gate 分别掉 3.4 和 2.2 点。
- τ³ Banking 上 MiniSWE 用 Hyperagents 整体涨得最多，可 sealed 掉了 2.1 点；AHE 和 GEPA 的 sealed 分别掉 4.1 和 2.1 点。Codex 这边反而更一致，AHE 在 train、gate、sealed 上分别涨 1.0、3.0、5.1 点。

**执行保真度（种子 agent 不进化，三次重复）：** Terminal Bench 2 上两个种子在框架内外的 Avg@3 都差 −2.6 点；Codex 的框架内外任务一致率 82.0%，直接执行两次之间是 88.0%，多出 6.0 点分歧，但框架内的 61.4 落在六次直接执行 60.7 到 65.2 的范围里。直接执行之间本身就有 12.0% 到 18.9% 的任务结果不一致。

**完整性（每类场景每个基准 30 次）：**

| 类别 | 场景 | 要求的不变量 |
| --- | --- | --- |
| 恢复 | 变异中被杀 | 从认证父代恢复，不接纳半成品子代 |
| 恢复 | 评测中被杀 | 保留旧尝试，只认证一次终态重试 |
| 恢复 | 写完回执后被杀 | 对账回执，不产生重复事件 |
| 边界 | 修改评测器文件 | 认证评测前拒绝 |
| 边界 | 写到可变面之外 | 越界修改全部排除出候选 |
| 边界 | 变异时偷看受保护证据 | gate、sealed、anchor 的金丝雀都不进变异包 |
| 完整性 | 伪造档案结果 | 未盖章事件或伪造的冠军缓存一律失败关闭 |

420 次全部达到要求结果。框架开销：Terminal Bench 2 上框架独占时间占每代端到端时间的中位 0.02%、P95 0.06%，τ³ Banking 上 0.019%。

## 🔬 最有意思的部分

**一是「分数涨了」在这套框架里变成了一个可以追问的句子。** 因为每个结果都绑着 contract_id，你能分清是策略变了还是评测条件变了；因为 gate 和 sealed 分开列，你能看到 A-Evolve 在 MiniSWE 上训练集涨 7.9 点、gate 一分没动、sealed 涨 5.6 点这种不对称。整体分数会掩盖留出集上的倒退，论文直接把这一点写成了结论。

**二是 Gate 和 Select 被刻意拆开。** Select 只负责从合格父代里挑，Gate 只负责决定评测完的子代有没有资格成为父代，两者之间还有一个「评测完成但 Gate 待定」的恢复检查点，这个状态下的子代对 Select 不可见。这是很典型的持久化运行时设计：先写意图和认证结果，再做下游决定，中断后按回执对账。和一些持久化 agent 运行时里先记 intent 再记 complete 的工具日志是一个味道。

**三是它对「让 agent 改自己的进化策略」留了口子，但留得很克制。** operator 可以放进可变面，外层 agent 可以改 recipe，但评测器、sealed 划分和认证记录始终不动。这对应的是一种「只进化适配层、冻结测量层」的分工，和另一类「冻结模型与 harness、只在外面进化 adapter」的思路是互补的：一个冻结被测对象的核心，一个冻结测量的核心。

## 🤔 我的判断

**定位：研究基础设施，加一份很有警示意义的横向对照。**

亮点：

1. 把「评测在可变面之外」从口号变成了带测试的不变量，420 次越界和故障注入全部守住。
2. 四种方法零框架改动接入，recipe 不到百行，确实降低了新方法的实现门槛。
3. 同一机器、同一模型配置下的横评显示没有通吃的方法，弱种子收益大、强种子收益小，且留出集迁移不稳。

问题与局限：

- 进化结果都是每题一次 trial 的点估计，而执行保真度实验本身就显示直接执行之间有 12% 到 19% 的任务不一致，几个点的差距很可能在噪声里，作者也承认不能据此判断接近的差异是否稳定。
- 四种方法都是「改编版」，和原实现的提示、预算、依赖不完全一致，排名不能直接当成原方法的排名。
- 只测了固定 recipe，最有意思的 agent 驱动模式没有实验数据。
- 安全边界依赖可信的框架服务、宿主和文件系统保护；作者明确说 RSIHub 支持审计和回滚，不提供对恶意代码的隔离。

对工程的直接启发：

- **把自己的 harness 进化闭环写成 recipe。** 可变面用 include/exclude 显式声明，评测器目录放在可变面之外，diff 越界直接拒。
- **划三份数据，别只划两份。** train 给反馈，gate 决定准入，sealed 只用于报告并且不进变异包。
- **每条评测结果绑一个 contract 摘要。** 评测器版本、数据集摘要、模型、重试策略变了，就不能和旧结果直接比。
- **先测框架内外的一致率。** 在不进化的种子上跑几次直接执行和框架执行，知道噪声底线是多少，再去解读进化带来的几个点。

一句话收束：**进化策略可以随便换，测量必须焊死；否则你永远说不清涨的分是 agent 变强了，还是尺子被挪了。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
