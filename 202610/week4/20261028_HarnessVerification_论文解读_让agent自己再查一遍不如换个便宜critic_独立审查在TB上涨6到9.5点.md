---
month: 202610
week: 4
date: 2026-10-28
type: 论文解读
slug: HarnessVerification
---

# 让 agent 自己再查一遍，不如换个便宜 critic：独立审查在 Terminal-Bench 上涨 6 到 9.5 点

你有没有过这种体验：给 coding agent 的系统提示里写了一大段「改完一定要跑一遍、看输出、看退出码」，它照样改完就宣布完成；你又加了个 stop hook，agent 一想收工就把它拽回来「再检查一下」，结果账单涨了一半，通过率只多了几个点，偶尔还因为「检查」把本来对的东西改坏了。

业内把这些手段笼统叫 **verification**：Claude Code 有完成时的 hooks，Codex 讲 self-review 和迭代，各家 harness 都在凭经验往里塞。问题是：**验证这件事在 harness 里到底该放在哪一步、由谁来做、查出问题之后怎么回到修改？花出去的算力值不值？**

康奈尔大学和阿里巴巴的 **Rethinking Verification in Agent Harnesses**（arXiv:2610.34577）把这个问题拆成一个四层框架，在七个前沿 coding 模型、两个 coding 基准加一个人评视觉基准上比了 17 种设计。看完我的感受：这是一篇非常「工程手册」味的对照实验，结论很朴素，**别让主 agent 反复自查，派一个新开上下文、只看任务和 diff、能进沙箱跑东西的便宜 critic**，但它给出的成本账让这个朴素结论变得有说服力。

## 核心摘要

论文沿 agent 生命周期把验证分成四层：L0 在开工前的初始指令里加验证要求，L1 在执行中插提醒（定期或每次改文件后），L2 在 agent 想停下时设完成闸门（要求复查、要求真跑一遍、或者干脆驳回「还没完成」），L3 在交付前请一个独立 critic 在全新上下文里检查成品并把发现回传给 agent 修。实验用 Qwen3.7-Max、Qwen3.8-Max、GLM-5.2、DeepSeek V4-Pro、DeepSeek V4-Flash、Kimi-K2.7-Code、MiniMax-M2.5 七个模型，Terminal-Bench 2.1 选了 31 题、SWE-bench Verified 选了约 30 题，每个条件每题三次 trial，critic 统一用 Qwen3.7-Plus。结果是：**L0 和 L1 收益不大且很看模型，但几乎不增加输出 token；L2 能涨得多一些，但驳回类设计成本到基线的 1.60 到 1.76 倍（TB）；L3 四种设计在 TB 上平均涨 5.9 到 9.5 点、SWE 上涨 3.9 到 6.6 点，总成本只有基线的 1.13 到 1.25 倍和 1.00 到 1.04 倍。** 在 18 题的 VisualBench 盲评上，L3 四种设计净偏好 +13.2 到 +23.8 点，而 L0、L1 是 -3.2，L2 只有 +2.7。再往上加审查轮数或并行 critic 数，TB 平均还能再涨最多 3.5 点。真正的价值定位是：把「验证」当成测试时扩展的一根独立轴，并给出了每层的成本收益对照。

## 论文信息

- **标题**：Rethinking Verification in Agent Harnesses
- **作者**：Zhengxin Zhang、Erkang Zhu、Yin Lin、Claire Cardie、Bolin Ding、Jingren Zhou
- **机构**：Cornell University；Alibaba Group（一作为阿里实习期间完成）
- **链接**：https://arxiv.org/abs/2610.34577 （v1，2026-10-25）

---

## 🎯 为什么这件事值得写

我们这几周归档的 harness 论文，不少都在讲「怎么改 harness」：RiseHarness 对照成败轨迹，HarnessRL 修 rubric 奖励，TraceTrial 把 harness 义务单独打分，VerifierBench 发现 agent 自己写的验证器拒错准、认对差。但「验证环节本身该怎么摆」一直是靠工程直觉：hooks、self-review、再跑一遍测试，各家各有说法，很少有人在同一套环境里横向比过。

论文给了一个让人很难反驳的动机数据：在 TB 基线 run 里，**只有 6.2% 到 13.6% 的编辑回合在下一次编辑或 run 结束前跟着一条可识别的检查命令**（附录 Table 7，按「之后回合」口径）。也就是说，agent 大部分时候是改完接着改，改完就交。

作者把设计空间组织成三个问题：

1. **什么时候介入**：开工前、执行中、想停时、交付前。
2. **谁来检查**：agent 自己，还是独立的 critic。
3. **怎么从检查走到修改**：提醒、复查、驳回，或者带着具体发现回去修。

## 🏗️ 机制：四层验证与 17 种设计

| 层 | 介入时机 | 谁检查 | 执行怎么继续 |
| --- | --- | --- | --- |
| L0 初始指令 | 初始化 | agent | 用初始验证指令引导检查和修改 |
| L1 执行时提醒 | 执行中 | agent | 收到提醒后检查最近改动、修正进行中的工作 |
| L2 完成闸门 | 想结束时 | agent | 收到复查请求或被驳回后继续自查修改 |
| L3 独立审查 | 交付前 | 独立 critic | critic 的发现回传给 agent 修 |

**L0** 只变一个量：验证指令放在 system 还是附在用户请求里。指令内容是让 agent 通过交付物真实的使用入口跑一遍、看输出、修缺陷；视觉任务还要渲染并操作核心交互。

**L1** 有三种：定期提醒（system 角色，首次编辑后每三回合一次）、改后提醒（system 角色，每次改文件后）、改后提醒（user 角色，不带自动化标记）。system 角色版本带一行 `[AUTOMATED SELF-VERIFY CHECKPOINT - not a message from the user]`，提醒都在工具结果之后、下一次模型调用之前注入，最多四次。

**L2** 的触发条件是「一回合没有工具调用」，变量包括：

- 要求的动作：Inspect changes（重读改动）、Run and check（真跑）、Reject completion（直接说「自动验证判定未完成，继续干」）、Recheck before finishing（用户口吻请再确认）；
- 消息角色和有没有自动化标记；
- 时机：early 变体在任务时限 80% 时就提前触发一次。

值得注意：**Reject completion 的文案只是一句「An automated verification pass has determined this task is NOT complete」**，并没有真的跑什么验证。

**L3** 的 critic 在全新上下文启动，拿到任务说明并能访问实时沙箱，可以自己跑检查，但**没有权限修改交付物**。两个变量：

- critic 看到的上下文：Changes（任务加 git diff）或 Trajectory（再加 agent 的完整执行轨迹）；
- 审查策略：Task-first（只按任务要求列清单）或 Claim-guided（把 diff 和轨迹当「审计地图」，把 agent 每一处「已测试」「可以了」都当作待证伪的假设）。

critic 的 prompt 很有意思：先列加权需求清单（3 核心、2 重要、1 加分），点名「看起来能过但其实错」的深层正确性属性；然后**通过真实入口运行**交付物，独立重算期望答案，不许调 agent 的内部函数，只允许在 `/tmp` 写一次性探针；最后只有亲手跑出的探针正面证明需求成立才判 done。单轮设计里，critic 认可就结束；有异议就把发现交给 agent，agent 的下一次交付直接被接受，不再复审。

成本按各模型单价分开计价，包含 coding agent、critic 和后续修改。

## 🧪 关键证据

**平均绝对准确率（基线 → 设计）：**

| 层 | 设计 | TB | SWE |
| --- | --- | --- | --- |
| L0 | 验证指令 System | 55.1 → 58.4 | 70.9 → 74.9 |
| L0 | 验证指令 User | 55.1 → 56.3 | 70.9 → 69.8 |
| L1 | 定期提醒 System | 55.1 → 58.9 | 70.9 → 74.1 |
| L1 | 改后提醒 User | 55.1 → 60.4 | 70.9 → 70.6 |
| L2 | Reject completion 自动 | 55.1 → 59.1 | 70.9 → 76.2 |
| L2 | Reject completion User / early | 55.1 → 61.1 | 70.9 → 77.2 |
| L3 | Task-first Changes | 55.1 → 62.0 | 70.9 → 77.5 |
| L3 | Claim-guided Trajectory | 55.1 → 63.5 | 70.9 → 77.0 |
| L3 | Task-first Trajectory | 55.1 → 64.6 | 70.9 → 74.9 |

**分层结论：**

- **L0**：system 角色在 TB/SWE 上 +3.3/+3.9，user 角色 +1.2/-1.1。差异很看模型，MiniMax-M2.5 在 SWE 上两种都涨（+12.6/+8.0），Qwen3.8-Max 在 SWE 上两种都跌（-2.3/-2.3），却在 TB 上涨 +8.0/+4.0。成本 1.08/1.03 倍（TB）和 0.94/0.93 倍（SWE）。
- **L1**：定期和改后提醒（system）效果相近，TB +3.8 对 +3.5，SWE +3.1 对 +3.6；改后提醒换成 user 角色，TB 涨到 +5.4，SWE 掉到 -0.3。**输出 token 反而少了**：TB 降 14% 到 25%，SWE 降 7% 到 10%。Qwen3.7-Max 在 TB 上用改后提醒（system/user）涨 +7.1/+14.3，同时输出 token 少 33%/28%。
- **L2**：自动驳回 +4.0/+5.3，Inspect changes 只有 +0.7/+0.5，自动 Run and check +2.6/-0.5。提前触发让用户驳回从 +2.1/+4.1 升到 +6.0/+6.2，是 L2 里最好的。但驳回类设计成本 1.60 到 1.76 倍（TB）、1.48 到 1.56 倍（SWE）；早期用户复查成本较低，1.13/1.20 倍。用户驳回之后，各模型中位数还要再走 12 到 19 步，而 Inspect changes 之后只有 4 到 6 步。
- **L3**：四种设计全部正收益，TB +5.9 到 +9.5，SWE +3.9 到 +6.6，成本 1.13 到 1.25 倍和 1.00 到 1.04 倍。**和最强的 L2 比，最强的 L3 在 TB 上涨得更多（+9.5 对 +6.0），SWE 上相当（+6.6 对 +6.2），两边都更便宜。** Kimi-K2.7-Code 在 SWE 上的四种 L3 都涨 13.8 到 20.7 点。

**VisualBench（18 题，人评盲测，净偏好）：** L0 和 L1 都是 -3.2，L2 早期复查 +2.7，L3 Changes +21.2，Changes 加强制视觉检查 +23.8，Trajectory +13.2，Trajectory 加视觉检查 +21.6。两个 Changes 设计在七个模型上全部正偏好，Trajectory 设计仍有模型是负的。GLM-5.2 很典型：L0 在两个 coding 基准都涨，VisualBench 上却是 -8.3。

**扩展审查：** TB 上一个 critic 一轮是 62.7%，平均 critic token 38.2k；两轮 65.6%（79.5k），两个 critic 64.4%（69.7k）；四轮 66.1%，四个 critic 65.8%。四个 critic 比一个多 3.1 点，总成本 1.33 倍；**四个 critic 先汇总意见再让 agent 改一次，接近四轮的准确率，总成本少 19%。**

诚实泼冷水：

- 题量不大，TB 31 题、SWE 约 30 题，每个条件三次 trial，很多单格差值是一两题的量级，正负号在模型之间来回跳。
- critic 固定用 Qwen3.7-Plus，评测的七个模型全部通过阿里云 DashScope 调用，没有换 critic 家族的对照。
- VisualBench 是内部 18 题，评审是公司内部数据团队。
- 单轮 L3 的协议是「critic 有异议，agent 改完就直接收」，并不保证修改真修对了。

## 🔬 最有意思的部分

**一是「给 critic 看完整轨迹」不一定更好。** 直觉上 critic 看到 agent 的全部推理会更容易抓 bug，但三个基准上 Trajectory 都没有稳定胜过 Changes，VisualBench 上 Changes 反而明显更好（+21.2 对 +13.2），而 Trajectory 设计的 critic 成本在 TB 和 SWE 上高 29% 到 43%。作者的推荐是**默认只给 critic 任务需求和 diff**。我的理解是，轨迹里充满了 agent 的自我说服，critic 读多了容易被带进同一个思路；只看 diff 反而逼它从任务出发去重新跑。

**二是一句假的「验证判定未完成」也能涨分。** Reject completion 的文案没有附任何具体发现，就是一句「自动验证判定未完成，继续干」。它在 SWE 上平均涨 +5.3（自动版），提前触发的用户版涨 +6.2；用户驳回后 86% 到 96% 的 run 会去改文件。这说明很多 agent 的「完成」声明本身就偏早，但也说明这种闸门是在用额外步数换分：中位数多走 12 到 19 步，成本到 1.5 倍以上。案例里也有反面：Kimi-K2.7-Code 在 `git-multibranch` 上被提前复查后，做部署测试时把测试内容 commit 并 push 到被服务的分支，没有清理，原本能过的基线反而挂了。**闸门只是给了修的机会，agent 还得找得到真正的缺陷，并且保住一个合法的最终状态。**

**三是 critic 赢在「跑了 agent 没想到的那个用例」。** `regex-log` 上 MiniMax-M2.5 自测后宣布完成，critic 测了日期出现在 IP 之前的情况，暴露漏匹配；`cobol-modernization` 上 Qwen3.7-Max 自测了合法和非法交易，critic 去测算术下溢，发现 Python 写出了负号而 COBOL 无符号字段没有，记录格式因此不对。两例里 critic 交回去的都是**一个可执行的失败**，不是一段评语。

## 🤔 我的判断

**定位：工程对照实验，测量的是 harness 层验证设计的成本收益曲线，不是新方法。**

亮点：

1. 四层框架把「何时、谁、怎么修」三个变量拆开了，每一层的结论都能直接落到 harness 配置项上：system prompt 一小段、改后提醒、stop hook、独立 critic。
2. 成本口径完整，包含 critic 和后续修改，并且用不同模型单价分别计价，L3「涨得多还更便宜」这个结论因此站得住。
3. critic prompt 本身就是一份很好的「反自我说服」规范：把 agent 的每一处成功声明当假设去证伪，只认亲手跑出的观察，不许复用 agent 的内部函数。

问题与局限：

- 样本量偏小，单模型单设计的差值噪声很大，更适合看平均趋势，不适合拿某一格去做决策。
- 只有一个 critic 模型，且 critic 与部分被测模型同属 Qwen 家族，critic 的独立性和可迁移性都还没测。
- 单轮 L3 对修改结果不再复审，扩展实验里加轮数和 critic 数能再涨，但成本也线性上去了。

对工程的直接启发：

- **把 stop hook 从「让主 agent 再查一遍」换成「派一个只读 critic」。** 新上下文、任务加 diff、能进沙箱跑探针、不能改交付物，有异议就带着可执行的失败回传。这和最近说的「LLM 产出之后接独立闸门」是同一个方向，而且它天然适合挂在 harness 会话状态机的「待交付」状态上。
- **改后提醒值得默认开。** 它几乎不花钱，TB 上还减少了输出 token；如果有视觉产物，提醒里一定要包括渲染和交互检查。
- **并行 critic 先汇总再修，比串行多轮便宜。** 这对 subagent 路由很有参考价值：多个便宜 reviewer 并行跑，最后合并成一份发现清单交给主 agent，比主 agent 反复被打回要省。

一句话收束：**验证的算力该花在一双新眼睛上，而不是让同一双眼睛再看一遍。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
