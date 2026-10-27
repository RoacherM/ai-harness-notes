---
month: 202610
week: 4
date: 2026-10-27
type: 论文解读
slug: TraceTrial
---

# 任务做对了，不代表 harness 没出错：三成 run 交付全对，但至少一条 harness 义务没守住

你有没有过这种体验：同一个模型，换到 Claude Code 里跑得顺，换到 Codex 或 OpenClaw 里就时灵时不灵，可榜单上只给你一个 pass rate，你根本说不清差在哪。更难受的是另一种情况：任务明明过了，你翻日志才发现工具输出被截断后 harness 没给出恢复线索、子 agent 的完成信号根本没回来、命令退出码 23 只在文本里出现而结构化字段写着 `error: false`。模型凭运气或者凭推理补上了，交付物是对的，但 harness 那一层其实已经坏了。

业内一般把这一层叫 **agent harness**：构造上下文、暴露工具、搬运调用和结果、维护会话状态、执行权限控制。问题是：**现有 benchmark 测的是模型加 harness 的整体结果，能不能把「harness 有没有守住它该守的义务」单独测出来？**

中科院自动化所、中关村学院、中科院信工所、清华、北理工等团队（TraceLite AI）的 **Trial by Trace: Benchmarking AI Agent Harnesses for Reliability and Failure Attribution**（arXiv:2610.32285）做的就是这件事。看完我的感受：这是一篇非常「审计味」的测量基础设施论文，它最有价值的不是排名，而是那套「证据从哪个边界来、能支持到哪一级结论」的纪律。

## 核心摘要

TraceTrial-Bench 把原生 harness 本身当成被测系统，模型、任务、环境、预算是受控输入，评测器和采集机制是测量仪器。它沿用已有的 harness 分类法，把可靠性拆成七个维度：Context、Execution、Governance、Lifecycle、Observability、Tools、Verification，下设 40 个机制级属性，构造 95 个专家级、面向机制的任务（46 个单轮、45 个固定多轮、4 个模拟器驱动），每个任务都同时有 Harness 层评分表和 Agent 层评分表，并配可执行或基于状态的真值。四个模块撑起整套流程：Trajectory Tracing 把各家原生日志归一成 Trace IR，Multi-source Evidence Acquisition 按评分项用声明式选择器从轨迹、工作区、模拟器账本、探针里取有界证据包，两层评估先定 Agent 分再定 Harness 分，Failure Attribution 用任务里预先声明的机制链把失败的 Harness 项和失败的 Agent 项连起来。在 Claude Code、Codex、DeepSeek Harness、OpenClaw 四个 harness、九个模型、95 个任务共 3,420 次 run 上，**Harness 宏平均从 Codex 的 59.8% 到 OpenClaw 的 72.0%，相差 12.2 个点；Agent 任务成功率最高的是 Claude Code（44.0%），Harness 得分最高的却是 OpenClaw；1,065 次 run（31.1%）Agent 层全对但至少有一条 Harness 项失败；两层全对的只有 286 次（8.4%）。** 真正的价值定位是：只看结果的评测无法刻画 harness 可靠性，而这套 benchmark 给出了可审计、可复算的替代方案。

## 论文信息

- **标题**：Trial by Trace: Benchmarking AI Agent Harnesses for Reliability and Failure Attribution
- **作者**：Zhiyong Cao、Hengjie Zhu、Yongxin Wang、Xuchen Li（核心贡献者）等 23 人，通讯作者 Baihua Xiao、Sicheng Zhou
- **机构**：TraceLite AI；中国科学院自动化研究所、中关村学院、中国科学院信息工程研究所、清华大学、北京理工大学
- **链接**：https://arxiv.org/abs/2610.32285 （v1，2026-10-23）
- **代码**：https://github.com/TraceLite-AI/TraceTrial-Bench

---

## 🎯 为什么这件事值得写

过去两周我们归档了一串 harness 方向的论文：HarnessScaling 说只在一个脚手架里练 RL 会偏科，IQuest-Q1 说多 harness 一起练要先归因再算奖励，RiseHarness 和 AgentS4 在讨论怎么自动改 harness。它们都隐含一个前提：**我们知道 harness 哪里坏了。** 但实际上大多数 benchmark（Terminal-Bench、SWE-Bench Pro、τ-bench）只看最终答案、测试结果或环境状态，这个分数是模型和 harness 的合力。

论文用一张图把缺口讲得很清楚：任务成功可能掩盖了丢失的上下文、误导性的工具结果、违反的生命周期约束；任务失败也可能发生在 harness 执行完全忠实的情况下。即便固定模型，只比结果也分不清是哪种 harness 行为造成了差距。

所以作者要回答三个独立的问题：

1. agent 有没有交出要求的产物和动作？
2. 原生 harness 有没有保住任务要求的信息、执行语义和生命周期行为？
3. 观察到的 agent 侧失败，是否符合任务事先声明的「harness 失败传导到 agent」的机制？

## 🏗️ 机制：边界、证据、两层分、四级结论

**一次 run 的记录**被定义成

$$
R_{h,m,t,r}=(\tau,\Delta W,G,V_H,V_A,L_{\mathrm{act}})
$$

其中 $\tau$ 是归一化事件序列，$\Delta W$ 是相对冻结基线的工作区变化，$G$ 是仪器准入状态，$V_H$、$V_A$ 是两层的评分项判决，$L_{\mathrm{act}}$ 是被激活的候选机制链。

**观察是分边界的。** 一个「工具结果」至少有五种不等价的表示：工具进程真实产出的结果、harness 变换后的结果、模型实际看到的载荷、原生会话存档里的记录、评测器看到的归一化事件。论文规定每类结论只认对应的权威来源：环境或模拟器状态决定「实际发生了什么」，变更的产物决定「最终交付了什么」，原生的模型可见记录决定「模型能看到什么」，provider 请求证据只对它截获的结构化工具声明有效。正因为同时保留了「外部进程超时了」和「模型看到的结果里没提超时」，harness 忠实度这个评分项才成立。

**七个维度与任务数：**

| 维度 | 评估焦点 | 任务数 |
| --- | --- | --- |
| Context | 跨边界的信息保留与披露 | 15 |
| Execution | 进程语义、状态、结果传递是否忠实 | 27 |
| Governance | 审批、护栏、密钥脱敏、信任边界 | 9 |
| Lifecycle | 状态转移、恢复、幂等重试 | 19 |
| Observability | 模型可见诊断、可回放日志、溯源 | 7 |
| Tools | 工具接口有效、传输忠实、可发现 | 17 |
| Verification | 完成声明与权威结果一致 | 1 |

任务触发条件很工程化：超大输出需要恢复、非零退出码配上「看起来成功」的产物、挂起的作业、跨会话边界的事实、只能通过交互通道拿到的私有值。

**跨 harness 适配器只翻译，不替换。** 原生的提示构造、工具传输、上下文策略、权限和会话行为都保留；允许表示层归一，但禁止引入新能力、加强模型可见证据或改变任务判定条件。Codex 读 JSON 事件流，Claude Code 合并 stream-json 输出与会话记录，OpenClaw 读 JSON 输出加会话 JSONL，DeepSeek Harness 读持久化的 JSON-RPC 通知流。调用和结果用原生 call id 关联，未匹配、延迟、重复的记录**明确保持不关联**，不做启发式配对。

**证据包是确定性的。** 每条语义评分项声明可选的事件类型、字段、字面匹配条件和最大返回条数；长字符串保留固定边界片段和关键词窗口，大集合追加显式的裁剪记录，并报告扫描数、匹配数、返回数和截断标志。全局上限和每项上限都固定在存档的评分表里，**裁判模型在评估中无法改变它们**。

**两层评分的顺序。** 先过仪器准入（容器生命周期、存档完整性、轨迹可评分、探针完整、私有 oracle 可用），准入不是第三个分数，也不能弥补任何一层。之后能用确定性规则判的项先判并锁定，只把没有 oracle 的项交给语义裁判。Agent 层先评、以变更产物为主证据；Agent 结果锁定后再评 Harness 层，Agent 判决只作为下游上下文，不能当作 Harness 失败的证据。选择器返回空证据时该项记 0，并明确记录「在声明的证据契约下未能确立满足」，但这本身不证明 harness 有缺陷。

Harness 语义项有四个状态：0 不支持、1 支持、2 未被触发（机制不存在或 run 没走到那条路径，按 Agent 结果解析为通过或失败）、3 机制存在且被触达但 harness 自身故障（直接判 Harness 失败）。单次 run 得分为

$$
S=50\frac{p_H}{n_H}+50\frac{p_A}{n_A}
$$

只有 $S=100$ 才算组合通过，但连续分、逐项判决和归因都保留。

**归因的四级结论。** 评分项判决 → 经权威来源核对后的确认缺陷 $D_H$ → 激活的候选机制链 → 经人工复核的传导结论 $L_{\mathrm{val}}$。候选链 $\ell=(h_\ell,a_\ell,\mu_\ell)$ 在执行前由任务声明，运行时只有当 Harness 端点失败、Agent 端点失败、激活条件满足时才激活：

$$
L_{\mathrm{act}}(R)=\{\ell\in\mathcal{C}^{\mathrm{run}}_t\mid h_\ell\in F_H,\ a_\ell\in Z_A,\ q_\ell(R)=1\}
$$

升级为已验证传导，还要人工确认可观察的中间失真、事件顺序和竞争解释。论文反复强调，运行时标签 `harness_caused` 只表示候选链激活，不是因果结论；`agent_caused` 也不代表 harness 无缺陷。还有一个「测量旁路」细节：比如 agent 用包装命令吞掉了被测命令的退出码，目标属性变得不可观察，这类 Harness 零分在调整集里会被剔除。

## 🧪 关键证据

固定裁判为 deepseek-v4-flash，九个模型组是 DeepSeek-V4-Pro、Gemini 3.7 Flash、GLM-5.3、GPT-5.6-Sol、Grok-4.6、HY3、Kimi K3、Opus 5、Qwen3.8-Max。每个 harness 855 个格子，全部 3,420 个格子都有最终评分，无缺失插补。

**四个 harness 总表（%）：**

| Harness | Agent Pass | Agent Score | Harness Micro | Harness Macro | 两层全对 | 综合分 | 平均成本($) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Claude Code | **44.0** | **75.9** | 71.5 | 71.2 | 9.1 | **73.5** | 0.436 |
| Codex | 35.8 | 70.7 | 60.2 | 59.8 | 2.8 | 65.2 | 0.595 |
| DeepSeek Harness | 40.0 | 73.3 | 69.5 | 68.8 | 9.8 | 71.0 | 0.446 |
| OpenClaw | 38.2 | 71.7 | **71.9** | **72.0** | **11.7** | 71.8 | 0.633 |
| 全部 | 39.5 | 72.9 | 68.3 | 67.9 | 8.4 | 70.4 | 0.528 |

几个值得记住的数：

- 全体 Agent 项通过 14,979/20,016（74.8%），Harness 项通过 19,242/28,188（68.3%）。
- Harness 宏平均的跨度 12.2 点，比 Agent 成功率的跨度 8.2 点更大。
- **Codex 在每一个模型组里 Harness 宏平均都是最低的**；OpenClaw 在六个模型上最高，Claude Code 在 GLM-5.3、GPT-5.6-Sol、Grok-4.6 上最高。
- 同一模型换 harness，Agent 成功率差距很大：Grok-4.6 从 OpenClaw 的 33.7% 到 Claude Code 的 55.8%，差 22.1 点；Opus 5 从 Codex 的 30.5% 到 Claude Code 的 53.7%，差 23.2 点。

**按维度看（Harness Micro，%）：**

| 维度 | Claude Code | Codex | DeepSeek | OpenClaw | 合并 | Agent Pass |
| --- | --- | --- | --- | --- | --- | --- |
| Context | **74.5** | 62.0 | 64.3 | 61.6 | 65.6 | 45.2 |
| Execution | 64.7 | 51.5 | 65.3 | **77.1** | 64.6 | 49.2 |
| Governance | 72.3 | 74.2 | 68.8 | **80.5** | 74.0 | 27.2 |
| Lifecycle | 71.3 | 53.0 | **71.7** | 65.9 | 65.5 | 43.6 |
| Observability | **88.7** | 76.5 | 86.8 | 85.0 | 84.3 | 43.3 |
| Tools | 71.5 | 65.6 | 71.5 | 70.2 | 69.7 | 21.4 |

Execution 合并后最低（64.6%），但 Agent 成功率反而是多任务维度里最高的（49.2%）；Tools 维度 Harness 69.7%，Agent 成功率只有 21.4%。40 个属性里没有一个在所有组合下达到 100%，网络限制最低只有 38.9%。

**失败状态分布：** 286 次两层全对，1,065 次只有 Harness 短板，369 次只有 Agent 短板，1,700 次两层都有短板。两层都失败的 1,700 次里，842 次（49.5%）至少激活一条候选链，共 1,633 对端点；2,069 次 Agent 失败的 run 里，1,773 次（85.7%）至少还有一个 Agent 失败项没有被任何链覆盖。

诚实泼冷水：每个格子只保留一次选中的执行，不估计重复稳定性；任务修订、运行设置、裁判输出上限（4,096 / 8,192 / 32,000 token 三档混用）不完全一致；作者自己也说不估计选择器召回、裁判一致性和候选链精度。Verification 维度只有 1 个任务，那一行数字基本不能用来比较。

## 🔬 最有意思的部分

**一是附录里的五个配对轨迹，几乎每个都是一条可以直接写进 harness 规范的条款。** 同样用 Opus 5：

- 截断恢复：Claude Code 在输出过大（5.7MB）时仍把 `SNAPSHOT_ID` 和 `TOTAL_ESCALATED=12` 留在可见结果里，agent 据此分页取回，Agent 5/5、Harness 10/11；OpenClaw 只告诉模型「还有 136,935 个字符被截断，需要的话用更窄参数重跑」，agent 只能用 `--snapshot UNKNOWN` 去取，被拒，Agent 0/5、Harness 5/11。**截断提示不够，可见结果必须保留恢复所需的信息。**
- 类型化错误：命令打印 `STATUS: OK` 却以退出码 23 结束。OpenClaw 的后置观察器记录 `exit_code: 23, error: true`；DeepSeek Harness 只在文本里写 `[exit code: 23]`，结构化字段是 `exit_code: null, error: false`。两边 agent 报告完全一样都对（4/4），但 Harness 分分别是 7/7 和 4/7。**同样正确的报告可能掩盖不同的失败通道。**
- 异步委派：Claude Code 派出子 agent 后收到自动完成通知，拿到 sum 和 nonce；Codex 的子 agent 回复「payload 里看不到任务细节」，协调者只能写 `delegation: "pending"`。**派发被接受、任务完成、可发布是三个不同事件。**

**二是「Harness 分」和「Agent 分」在不同维度上的差距方向相反。** Execution 和 Lifecycle 上 Agent 分明显高于 Harness 分（78.0 对 64.7，77.4 对 65.8），说明模型在替 harness 兜底；Tools 上反过来（53.2 对 66.8），工具层看起来没大毛病，但模型就是做不对。这意味着「harness 好不好」和「这套 harness 适不适合你的任务」是两个问题，论文的建议是按目标任务的可靠性需求选 harness。

**三是它把裁判关进了笼子。** 确定性规则先判并锁定；裁判只看声明式选择器取出的有界证据包；Agent 判决不能当作 Harness 失败证据；每次裁判调用的提示、回复、解析分和理由都存档。这和最近大家常说的「LLM 评审之后接确定性闸门」是同一类思路，只不过这里是反过来：**先过确定性闸门，再让 LLM 在一个被限定的证据窗口里判。**

## 🤔 我的判断

**定位：测量基础设施，并且是目前我见过把 harness 当被测系统讲得最严谨的一篇。**

亮点：

1. 证据权威按边界划分，并且写成了规则，「模型看到了什么」和「实际发生了什么」不再被混为一谈。
2. 两层评分加四级结论，把「共现」和「传导」严格分开，49.5% 这个激活率本身就在提醒大家：很多失败并不能简单甩锅给 harness。
3. 跨四个真实 harness、九个前沿模型，而且适配器不替换原生行为，比在自研脚手架里做消融更接近工程现实。

问题与局限：

- 每格单次执行，12 点左右的差距是否稳定无法判断；对 harness 设计选择做因果解读要很小心。
- 任务覆盖不均匀，Verification 只有 1 题、Observability 只有 7 题。
- 语义裁判只用了一个模型，且输出上限不统一，裁判一致性没有估计。
- 机制链需要任务作者预先声明，覆盖不到的失败就落进「未关联」，85.7% 的 Agent 失败 run 里都还有未被链覆盖的失败项。

对工程的直接启发：

- **给自己的 harness 定一份 Trace IR。** 至少统一 `tool_call` / `tool_result`、`context_compaction`、`subagent_lifecycle`、`policy_event` 几类事件，并保留原始记录，后面所有分析都建立在这上面。
- **结构化错误字段要真的结构化。** 退出码、超时、截断大小写进字段，不要只写进给模型看的文本。
- **截断时把恢复钥匙留下。** 快照 id、总数、分页游标比一句「输出过长」有用得多。
- **把 H-only 的 run 当成改 harness 的第一批样本。** 这 1,065 次是「模型兜住了但 harness 有短板」的案例，换个弱一点的模型就会暴露出来。论文在未来方向里也提到可以用它做 trace 引导的 harness 进化闭环，这正好接上 RiseHarness、RSIHub 这一路。

一句话收束：**pass rate 是模型和 harness 一起交的卷子，想知道 harness 自己考了几分，得先把证据按边界拆开。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
