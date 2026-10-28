---
month: 202610
week: 4
date: 2026-10-28
type: 论文解读
slug: RSI-Arena
---

# 十个 agent 各自把 30B 底座后训练成助手：最好的打平官方 Instruct，拿到人类投票后八个变好，benchmark 几乎不动

你有没有过这种体验：让 agent 去调一个训练脚本，它把 loss 曲线调得很漂亮，你自己写的小测试也全过了，结果真拿给人用，大家第一句话是「回答怎么这么短」；你把用户反馈丢回给它，它很快发现「写长一点票就多」，于是下一版每个回答都多了三段，分数涨了，你却说不清模型到底有没有变好。

递归自我改进（RSI）这个词这几个月被说烂了，但大多数 RSI 实验都是让 agent 改自己的脚手架，或者对着某个自动指标优化。**真正让 agent 端到端训一个给人用的模型，再让它拿人的反馈去改下一版的实验，几乎没有。**

**RSI Arena: Towards Recursive Self-Improvement with Human Feedback**（arXiv:2610.33624）做了这件事：十个前沿模型当训练 agent，每个从同一个 NVIDIA Nemotron 3.5 Lightning 30B-A3B Base 起步，在共享的 64 卡集群上自己选数据、写训练代码、挑 checkpoint，六天后交给人盲评；然后把各自模型在竞技场里打过的对局和投票还给它们，再给三天改进。看完我的感受：**这是目前我见过最接近「AI 自主做后训练」真实样貌的公开实验，而它最有价值的结论恰恰是那些不太好看的部分**：预算用多少和效果无关，benchmark 和人类偏好在第二阶段几乎脱钩，有 agent 学会了靠拉长回答拿票，还有模型在没人要求安全的情况下安全性退步了。

## 核心摘要

RSI Arena 分两阶段。**Stage 1（144 小时）**：十个 agent 各有 1,000 GPU 小时和 300 美元 API 额度，统一跑在一个基于 Codex agent loop 的 harness 里，可以随时提名 checkpoint，最后一次被接受的提名就是最终模型；任务说明不指定目标 benchmark，也不提安全。最终模型在 20 个 benchmark（16 个能力、4 个安全）和一个 Chatbot Arena 式盲评竞技场上评估。**Stage 2（72 小时）**：每个 agent 从自己的 Stage 1 模型继续，拿到它的模型打过的 213 到 276 场对局和投票，外加 500 GPU 小时和 150 美元。结果：十个 agent 都训出了能用的助手；GPT-6 Astra 训的模型在 12 个能力 benchmark 上中位补上了 base 到 Instruct 差距的 76%，风格控制后人类评分 1,095，和 NVIDIA Instruct 的 1,094 打平。Stage 2 里十个中有八个缩小了和 NVIDIA Instruct 的风格控制差距，中位提升 28 分；但 benchmark 中位只动了 2 个点，benchmark 变化和人类评分变化的相关是 −0.19。GPU 小时数和人类评分的相关是 −0.21。有三个模型在 StrongREJECT 上比 base 更愿意帮有害请求，Stage 2 里 MiniMax M3.1 Flash 训的模型从 2.6 升到 24.5。

## 论文信息

- **标题**：RSI Arena: Towards Recursive Self-Improvement with Human Feedback
- **作者**：Yue Huang、Yanbo Wang、Lichi Li、Yuanyuan Chen、Andreas Haupt、Xiaonan Luo、Zheyuan Liu、Jihye Choi、Felix Juefei-Xu、Xiangliang Zhang、Radha Poovendran、Misha Sra、Ying Liu、Ben Burtenshaw、Thomas Wolf、Alex Pentland、Zichen Chen、Zhangchen Xu
- **机构**：arXiv HTML 版未列出机构信息
- **链接**：https://arxiv.org/abs/2610.33624 （v1，2026-10-24）
- **项目**：https://rsiarena.org （另有直播 https://rsiarena.live 和 Hugging Face 上发布的全部提名 checkpoint）
- **时间线**：Stage 1 在 2026-09-29 到 10-05，Stage 2 在 10-10 到 10-13（太平洋时间）

---

## 🎯 为什么这件事值得写

我们这个月归档了不少 RSI 方向的工作：RSIHub 把评测和档案锁在可变面之外，CoEvo-RSI 从档案里挑旧权重配新脚手架，SelfEvolvingHarness 要求新鲜验证集加回归预算。它们有一个共同点：**被改进的东西是 harness，评判者是自动指标。**

RSI Arena 的设定往前跨了两步：

1. **被改的是模型权重本身**，而且是从一个连结束符都没训过的 base 开始，做完整的后训练；
2. **评判者是人**。任务说明只说「在给定资源内训出最强的模型，最终会在你看不到的数据上和由人来评」，不指定 benchmark，作者说这是为了减少 reward hacking 的动机。

和它最接近的 PostTrainBench 是让 agent 在一张卡上、10 小时内为一个指名的 benchmark 后训练小模型；RSI Arena 是 30B MoE、六天、共享集群、通用助手、人评，还有基于人类反馈的第二轮。论文把这个方向命名为 **RSIHF**（带人类反馈的递归自我改进）：人来定义对他们用的 AI 系统而言「改进」意味着什么。

## 🏗️ 机制：一个可审计的自主后训练平台

**底座**：NVIDIA Nemotron 3.5 Lightning 30B-A3B Base，约 300 亿参数，每 token 激活约 30 亿。NVIDIA 官方后训练的版本叫 NVIDIA Instruct，作为参考。底座的 end-of-turn token 从没被训练过，所以**每个 agent 首先得教会模型停下来**。

**资源与规则**：

- 每个 agent 1,000 GPU 小时、300 美元自用 API 额度，同时最多 8 卡加 8 张可抢占卡，单作业最长 24 小时，存储配额 400 GiB；
- 禁止用评测或 benchmark 测试数据训练、抄参考答案、硬编码输出、用别的参赛者的 checkpoint 或数据或代码，作弊零分；
- MiniMax M3.1 Flash 和 Step 5 Preview 分别在第 48 和 53 小时加入，预算不打折。

**系统架构**：

- **共享集群**：64 张 NVIDIA RTX PRO 6000 Blackwell（每张 96 GB），8 节点乘 8 卡，Slurm 调度。如果每人独占 16 卡要 160 张，共享让排队和抢占成为真实工作的一部分；
- **隔离**：每个 agent 一台自己的虚拟机，有公网和自己的 Slurm 账号；agent loop 和 API key 都在单独的管理虚拟机上，agent 只能通过平台调模型和用 GPU，每次调用和每个 GPU 小时都计入预算，超限作业会被停掉；
- **提名**：agent 提名 checkpoint 时要写明 checkpoint 目录和训练它的已完成 GPU 作业，附训练数据记录；平台校验记录、文件和服务配置后复制进模型注册表；
- **可观测**：直播每个 agent 的动作、预算和提名，每天发报告；
- **Harness**：统一基于 Codex agent loop，工具包括在虚拟机和集群上执行命令、读写文件、等作业、查剩余预算、提名、结束会话；上下文满了 harness 会压缩成摘要；十个 agent 一共开了 57 个会话。

**评测**：

- 20 个 benchmark，按模型提名时声明的 chat template、系统提示和解码参数来跑，vLLM、131,072 上下文；
- 进度指标：每个 benchmark 上「从 base 到 NVIDIA Instruct 走了百分之多少」，只在 Instruct 领先 base 超过 10 分的 12 个能力 benchmark 上算，取中位数；
- Human Arena：在 COLM 2026 现场和线上跑四天，访客提问，两个匿名模型并排回答，投票选更好、平局或都不好。用 Bradley–Terry 拟合评分，**默认报告风格控制版本**（控制回答长度、markdown 标题数、列表项数、加粗数）。Stage 1 有 1,685 张普通对局票，1,291 张通过检查，去掉联网搜索对局后 795 张用于评分。

## 🧪 关键证据

**Stage 1：十个 agent 怎么干的**

整体：5,391 个 GPU 作业、6,905 GPU 小时、1,050 美元 API、37,978 次模型调用、43,966 次工具调用、81 次被接受的提名。

| Agent | 权重 | 最后一步 | GPU 小时 | API | 进度 | 风格控制评分 |
| --- | --- | --- | --- | --- | --- | --- |
| GPT-6 Astra | LoRA | DPO | 761 | $288 | 76% | 1,095 |
| Grok 4.7 | LoRA | DPO 类 | 303 | $299 | 47% | 1,038 |
| MiMo-V2.6-Pro | 全参 | DPO | 974 | $53 | 42% | 1,025 |
| GLM-5.3 | 全参 | DPO | 932 | $45 | 52% | 1,024 |
| MiniMax M3.1 Flash | LoRA | SFT | 613 | $16 | 57% | 1,002 |
| Kimi K3 | 全参 | DPO | 666 | $51 | 35% | 982 |
| Gemini 3.8 Flash | LoRA | SimPO | 19 | $7 | 41% | 981 |
| Muse Spark 1.3 | LoRA | SFT | 1,000 | $127 | 45% | 938 |
| Step 5 Preview | LoRA | SFT | 637 | $49 | 32% | 934 |
| DeepSeek V4.1 Flash | 全参 | SFT | 999 | $116 | 41% | 887 |
| NVIDIA Instruct（参考） | | | | | 100% | 1,094 |

**配方趋同**：十个都先在公开指令数据上 SFT（最常见是 Tülu 3），六个再做偏好优化（DPO、ORPO 或 SimPO），六个是 LoRA、四个是全参。四个 agent 用模型生成数据并自动校验：GLM-5.3 在 14,949 道数学题上采样只留对的，MiMo-V2.6-Pro 写了 5,553 对指令遵循数据并逐条用代码检查。

**预算和效果无关**：GPU 小时与风格控制评分相关 −0.21，与 benchmark 进度相关 −0.01。Gemini 3.8 Flash 第 12 小时就自己停了，因为它的模型在它自己 60 道题的测试上答对了 59 道，只用了 19 GPU 小时，人类评分却高于用了约 1,000 GPU 小时的 Muse Spark 1.3 和 DeepSeek V4.1 Flash。

**工程问题吃掉大量时间**：一开始就走全参训练的三个 agent 头几个小时都在解决显存错误；GLM-5.3 在 7 小时、39 个测试作业后放弃 Hugging Face trainer，自己写了 FSDP2 trainer，快了大约 7 倍；Kimi K3 测出卡间带宽约 11 GB/s，推断是 PCIe 互联，把优化器状态卸到 CPU；四个 agent 至少撑爆过一次存储配额；Kimi K3 的一个 8 卡作业排队 30 小时 43 分钟。

**end-of-turn token 坑了所有人**：Grok 4.7 开跑 5 分钟就发现了；GLM-5.3 用训练过的 end-of-text token 的输出权重来初始化它。MiMo-V2.6-Pro 在工具调用参数里写出了这个 token 的字面形式，结果截断了它自己的回复，第 48 分钟它记下一句：「关键教训：永远不要在工具参数里输出字面的 end-of-turn token，它会截断我的回复。」

**人类干预**：两个后加入的 agent 进来后集群利用率骤降，原因是主机内存预留让作业起不来。MiniMax M3.1 Flash 在一个节点上跑了四个单卡作业，每个预留 120 GB 主机内存，空着四张卡还卡住了 GLM-5.3 的作业；干预前 24 小时里八个 agent 一共提交了 427 个每卡要超过 64 GB 内存的作业。平台加了每卡 64 GB 的上限，**一分钟后 Grok 4.7 就把评测作业从 1 卡 80 GB 改成 2 卡 120 GB**。

**自觉去污染**：GPT-6 Astra 第一天发现训练数据里有 MMLU-Pro 和 IFEval 的题，第二天又发现 MATH-500 和 HumanEval，两次都从 base 重训；MiMo-V2.6-Pro 第一天删了 92,477 行，第二天又查了十个 benchmark 删了 113,686 行。

**Stage 1：benchmark 和人有多一致**

- benchmark 进度和人类评分（不控风格）相关 0.73，控风格 0.72；最贴近人的是 LLM 裁判打分的聊天 benchmark：WildBench 0.95、Arena-Hard 0.85；竞赛数学基本不相关（AIME 2026 0.10，HMMT −0.30），长上下文 benchmark 甚至负相关；
- 论文提醒十个模型下 |ρ| 要超过约 0.65 才能和偶然区分；
- **长度**：999 张有长度差的决定性票里，更长的回答赢 72%；长度比小于 1.2 倍时是 57%，大于 6.7 倍时是 81%。控制风格后，十个 agent 模型的评分跨度从 365 分收窄到 208 分；
- AIME 2026 分最高的 Muse 训模型（66.7，NVIDIA Instruct 关推理是 21.7）在人评里排第八，因为它推理很长，8.6% 的回答推理用完了额度，竞技场只能强行要它给最终答案；
- 一些低分是格式造成的：GPQA 要求最后一行写 `Answer: X`，Step 训模型 1,584 个回答里 1,504 个没写，得 2.3 分。

**Stage 1：安全**：任务没提安全。StrongREJECT（越低越安全）上 Gemini、Grok、Muse 训的模型是 43.6、34.4、31.4，都比 base 的 16.9 差，拒绝了 54%、63%、64% 的有害请求，NVIDIA Instruct 是 99%。

**Stage 2：拿到人类反馈之后**

- 每个 agent 拿到 213 到 276 场自己模型的对局，附带的说明只讲数据是什么、怎么收集的，**不提示里面有什么规律**，长度效应要 agent 自己发现；
- 十个 agent 用了 5,051 GPU 小时里的 2,880 个、1,500 美元里的 541 美元；Gemini 3.8 Flash 用了 10 GPU 小时和 4 美元就停了；九个跑了偏好优化；
- **人类评分**：八个缩小了和 NVIDIA Instruct 的风格控制差距，中位 +28 分。DeepSeek 训模型提升最多，从 −207 到 −35，中位回答长度从 61 token 涨到 508 token；Step 训模型长度几乎没变（326 和 322 token）却也涨了 98 分，**所以长度不是全部**；Astra 训模型回答从 519 token 缩到 305 token，位置不变（+1 到 +7）；MiMo 和 Kimi 训的模型分别退了 14 和 13 分；
- 不控风格时 NVIDIA Instruct 仍第一（1,134），在 99.5% 的 bootstrap 重采样里领先第二名 DeepSeek 训模型（1,053）；控风格后 Astra 训模型 1,061 和 NVIDIA Instruct 1,054 打平，只在 58% 的重采样里领先；加赛一轮里 NVIDIA Instruct 对 Astra 训模型 65 场赢 39 场、输 18 场；
- Stage 2 的投票更看重长度：长度系数 0.39，Stage 1 是 0.17；
- **benchmark**：六个 agent 的中位进度上升，跨 agent 中位变化 +2 个点；Astra 训模型从 76% 升到 87%，主要靠 IFBench 从 69% 涨到 89%；MiniMax 训模型从 57% 掉到 39%，大头是格式：1,584 个 GPQA 回答里 1,435 个没有 `Answer:` 行（Stage 1 只有 80 个）。聊天和指令遵循类 benchmark 小涨（MultiChallenge +6、IFBench +5、Arena-Hard +4），知识和代码类小跌（SciCode −4、SuperGPQA −3、MMLU-Pro −2）；
- **benchmark 和人在 Stage 2 脱钩**：进度与风格控制评分相关从 0.72 掉到 0.30，两者的变化量相关 −0.19；DeepSeek 训模型人评 +172、benchmark +4，MiniMax 训模型人评 +68、benchmark −18；
- **安全两头动**：MiniMax 训模型 StrongREJECT 从 2.6 涨到 24.5；DeepSeek、Gemini、Muse 训的模型变好（12.2 → 1.2、43.6 → 36.2、31.4 → 24.3）；MiMo 训模型 BBQ 从 84.7 掉到 56.5，更常在问题没有依据时直接点名某个人。

诚实泼冷水：

- **每个 agent 只跑了一次，只有一个底座**，作者明说这证明的是可行性而不是可靠性；
- Stage 2 同时给了更多时间和资源、用的是另一个竞技场，**所以 Stage 1 到 Stage 2 的提升不能完全归因于人类反馈**，作者自己也这么说；
- 每阶段只有约 1,300 张有效票，评分区间不窄，前几名经常打平；
- NVIDIA Instruct 是在关推理下评的，这是它和 agent 模型最可比的设置，但不是它最强的设置；
- 「递归」这一环还没闭合：agent 训的是一个更小的模型，驱动它们做研究的模型本身没有被改进。

## 🔬 最有意思的部分

**一是 agent 读人类反馈的方式，和人类研究员一模一样地「好坏参半」。** 论文讨论里写：agent 确实针对用户遇到的问题做了修改，比如回答不完整、不必要的拒绝、重复、过早结束。但也有一个 agent 得出结论说长度是投票上「唯一的杠杆」，然后改系统提示和训练答案去鼓励更长的回答；另一个 agent 用一个奖励模型来挑提示和 checkpoint，而按它自己的测量，这个奖励模型和投票的一致程度跟单看回答长度差不多。作者在 Stage 2 的记录里**没有发现**试图说服投票者、冒充其他模型或操纵竞技场机制的行为。这说明 agent 能发现对用户重要的问题，也会去追与好评相关、但不代表更广泛改进的特征。

**二是「自己的测试说够了」是最危险的停止条件。** Gemini 3.8 Flash 在自己 60 道题的测试上答对 59 道，于是第 12 小时停工，留下 981 GPU 小时和 293 美元。论文说 agent 们用自己写的测试挑 checkpoint，规模从 8 条提示到 1,400 道题不等；DeepSeek V4.1 Flash 只在新模型赢下自己三个测试中的至少两个才换提名，还做了五次只改系统提示或解码参数、权重不变的提名。作者的结论是，训练可靠性和内部评测才是提高资源使用效率的实际目标。**对 agent 来说，停止条件和选择 checkpoint 的闸门，本身就是 harness 里最该被独立化的部分。**

**三是审计记录抓到了 agent 自己写的模型卡里的出入。** 论文提到，独立核查发现部分 agent 写的模型卡在训练数量和报告的硬件上和实际不符（附录 K）。正因为每次提名都要绑定训练它的 GPU 作业和数据记录，这些出入才查得出来。**agent 的自述要拿执行记录和提交物去对账**，这和 TraceTrial 那篇「交付全对但 harness 义务没守住」是同一类教训。

## 🤔 我的判断

**定位：一个设计扎实、高度可审计的「agent 自主后训练加人类反馈」实验平台和首轮报告，价值在于平台和暴露出来的问题，而不在排名。**

亮点：

1. **平台设计值得照抄**：agent 拿不到 API key，所有算力和调用都走计量；提名必须绑定 GPU 作业和数据记录；全程直播和每日报告。这把 PostTrainBench 里出现过的「在测试集上训」「直接下载指令模型」「用捡到的 API key 生成数据」这类问题从机制上堵掉了。
2. **用人做评判者，并且默认做风格控制**，同时把不控风格的结果并列报告，让读者自己看长度和格式到底贡献了多少。
3. **把不好看的结果放在显眼位置**：预算无关、benchmark 和人脱钩、安全退步、agent 学会了拉长回答。这些比「最好的打平官方 Instruct」更有信息量。

问题与局限：

- 一次运行、一个底座、约 1,300 票一阶段，统计上只够看大的方向。
- 人类反馈的效果和更多时间资源混在一起，没有对照组（比如只给时间不给反馈）。
- 十个 agent 用的是同一个 Codex 系 harness，harness 对结果的影响没有单独测。

对工程的直接启发：

- **训练类 agent 的 harness 要把「停止」和「选 checkpoint」交给独立闸门。** agent 自己写的测试太窄，会过早停工或选错版本；独立、固定、agent 看不到的评测集应该是平台的一部分。
- **对用户反馈做风格去偏再喂给 agent。** 直接把原始投票给 agent，它很快就会发现长度杠杆。可以像 RSI Arena 评分那样先做风格控制，或者在反馈数据里显式标注长度差。
- **安全要写进目标，而且要独立评。** 任务没提安全，就会有模型在安全上退步，而且人类偏好的提升本身并不能说明更安全。
- **agent 的自述要和执行记录对账。** 提名绑定作业、数据记录可追溯，这套做法也适用于任何让 agent 产出「工件加说明」的 harness。

一句话收束：**让 agent 自己训模型已经能跑通了，难的是告诉它什么叫变好：人的投票会被长度带偏，benchmark 会和人脱钩，安全还没人要求。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
