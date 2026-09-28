---
month: 202609
week: 4
date: 2026-09-28
type: 论文解读
slug: GameArena
---

# 主观裁判太吵、静态榜又饱和：用胜负当地面真值的对战评测台

你有没有过这种体验：MMLU / GSM8K 一类静态题快被刷满，分差缩到噪声；转头上 Chatbot Arena / MT-Bench，人票与 **LLM-as-judge** 又带着啰嗦偏好、谄媚和标准漂移，同一对回答换个裁判结果就翻。DeepMind × Kaggle × Google Cloud OCTO 这篇 Game Arena（Kaggle Game Arena，arXiv:2609.31473）把评测锚回 **可判定的胜负**：模型在统一文本 harness 里头对头下棋、打扑克、玩狼人杀，用 Elo / BB/100 / 博弈论评价（GTE）做榜，而不是再请一个更强模型当「品味裁判」。看完我的感受：这和 Byron 关心的 **judge / evals / harness** 正好形成对照——[LLM-as-Judge](../week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) / [DIAL](./20260928_DIAL_论文解读_消掉位置偏差还不够_少量人工把多裁判偏好校准成人味.md) 管「主观偏好如何校准」，Game Arena 管 **能落地到地面真值时，就把裁判层整块卸掉**；共享 harness（非法动作重试、方差控制、信息不对称总线）本身也是评测基建，不只是又一个游戏 demo。

## 核心摘要

Game Arena 是一个可扩展的开放对战平台：统一 **文本状态 → 单动作** 协议，非法输出给有限次重试（棋类不泄合法着法列表；扑克二次非法回退到最保守动作），全程记录轨迹与推理痕迹。首批三个环境覆盖信息结构光谱——**Chess**（完全信息、确定性）、**Poker** HU-NLHE（不完全信息、随机）、**Werewolf** 8 人（不完全信息 + 非对称 + 自然语言）。评测刻意堆统计功率：每对棋 40 局（颜色平衡）、扑克每对 20,000 手（100 手 episode × 座位镜像 duplicate，十模型全互打合计约 **90 万手**）、狼人杀约 **3.15 万** 局并用 GTE 拆角色贡献。十个前沿模型（GPT-5.2 / o3、Claude 4.5 系、Gemini 3 Pro/Flash Preview、Grok 4 / 4.1 Fast、DeepSeek V3.2 等）API 默认采样、无工具无微调。核心反直觉结论：**跨游戏排名会分叉**——Gemini 系稳居 Chess / Werewolf 顶层，扑克却是 GPT-5.2 / o3 / Grok 4 领跑，Gemini 3 Pro 扑克甚至为负；「战略能力」不是单一标量。定位不是又一个固定题集，而是 **会随对手变强而变难的评测基础设施**，开源 harness、轨迹与可视化（https://www.kaggle.com/game-arena · https://github.com/google-deepmind/game_arena）。

## 论文信息

- **标题**：Game Arena: Strategic LLM Evaluation in Competitive Environments
- **作者**：Bovard Doerschuk-Tiberi、Yao Yan、Justin Chiu、Hann Wang、Timothy Chung 等（GDM / Kaggle / OCTO 大协作，† equal contribution）
- **机构**：Google DeepMind、Kaggle、Google Cloud Office of the CTO
- **链接**：https://arxiv.org/abs/2609.31473 · https://arxiv.org/pdf/2609.31473 · HF https://huggingface.co/papers/2609.31473 · 平台 https://www.kaggle.com/game-arena · 代码 https://github.com/google-deepmind/game_arena
- **观察时间**：2026-09-28（HF Daily Papers）

---

## 🎯 为什么这件事值得写

评测困局被作者拆成两条死路：**静态基准**易饱和、易污染；**偏好型动态评测**（Chatbot Arena、LLM-as-judge）新鲜，但主观、噪声大、标准漂移。游戏史早已证明第三路——Deep Blue / AlphaGo / Libratus / Cicero——用 **可检验结果** 衡量智能；近年 AgentBench、GTBench、PokerBench、SmartPlay、Game Reasoning Arena 等把 LLM 推进游戏，却多为固定数据集或小样本一次性实验，缺持续互打、跨环境 onboard、以及扑克级方差下的样本量。

对 Byron（judge-evals / harness / coding-agent）：日常闭环里「Rubric → 代码检查 → golden+holdout → 结构化 judge」仍然正确，但很多任务其实有 **程序可判定结果**（单测、编译、棋规、筹码）。Game Arena 的方法论提醒是——**只要地面真值够硬，就别把 LLM-as-judge 当成默认评测器**；把预算留给真正开放的主观面。反过来，它把「评测 harness」写得很具体：统一文本接口、非法动作策略、上下文截断、事件可见性总线——这些和 agent 脚手架里的 tool contract / retry / 信息隔离是同一类工程对象，只是场景换成了棋牌狼人。

## 🏗️ 机制：共享 harness × 三环境 × 域适配指标

**共享协议。** 每步给自然语言状态 + 历史，要求规定格式的单一动作；非法则有限重试，反馈刻意极简（棋类只说「不合法」，不解释原因、不给候选着）。目标函数写死：棋「下最强合法着」、扑克「最大化期望价值（默认 GTO，仅在可读出对手倾向时偏离）」、狼人「团队胜利优先于个人存活」。这避免模型自选「娱乐局 / 保本局」污染强度估计。

**Chess。** FIDE 规则 + 环境执法；FEN + 完整 PGN，输出 SAN；颜色平衡每对 40 局；失败重试上限后判负。另开 **Chess Opening**：从 Lichess 热门两步开局之一起步，压掉「只会西西里」的开局记忆。主指标 Bradley–Terry Elo（最低模型锚定 0）+ bootstrap 95% CI；辅以 Stockfish 路径与 rethink（非法着重试）诊断。

**Poker。** 单挑无限注德州，盲注 1–2、每手 100BB 重置。100 手为一 episode，期内完整历史可见且 **每手结束后亮双方底牌**，加速对手建模；整段用预洗牌 **duplicate**（换座位再打一遍同一牌序）降方差。每对 20k 手；全互打 ~900k 手。非法动作二次后 check/fold。主指标 BB/100 + 按 episode 的 block bootstrap。

**Werewolf。** 8 人：2 狼 / 1 预言 / 1 医生 / 4 民。夜私行动、日公开辩论与投票；事件总线按可见性过滤，插件化讨论/投票顺序。规则经自对弈消融校准（禁医生自救与连续护、平票不处决、发言/投票轮转），把村民胜率从 ~73% 压到 ~57%，逼出真正的说服与欺骗而非「预言公开 + 医生连护」单调策略。Harness 用 ReAct + JSON（私推理 / 公动作分离）、pyjson5 宽容解析、溢出时保留最近 75% 上下文。主指标不用原始胜率（角色运气混杂），而用 **GTE**（MvMvR 元博弈 + 最大熵相关均衡）拆角色贡献。

## 🧪 关键证据

**Chess Text Elo 分层清晰。** Gemini 3 Pro Preview ~1325、Flash ~1297 顶层；o3 ~1009、GPT-5.2 ~933 次层；Grok / GPT-5 mini 中下；Claude 4.5 系与 DeepSeek V3.2 垫底（DeepSeek 锚定 0）。Stockfish 路径显示差距主要在中局/残局累积，开局段接近；弱模型 rethink 集中在后半盘（常「被将却走出不解将」）。Chess Opening 排名结构几乎不变——说明 Text 榜不全靠单一开局记忆。

**扑克三档且排序大换。** GPT-5.2 +46.6 BB/100（对所有对手占优）、o3 +29.7、Grok 4 +27.1；Claude Opus/Sonnet 与 Gemini Flash 中档仍盈利；DeepSeek / Gemini 3 Pro / Grok 4.1 Fast / GPT-5 mini 为负（mini −94.9）。bootstrap CI 把 GPT-5.2 与中下档完全拉开。翻前风格差异大（偷盲 53%–98%、3-bet 频率两极），但宽开并不等于盈利——GPT-5.2 与 GPT-5 mini 都宽，结果差 140+ BB/100，说明翻后决策质量才是分水岭。

**狼人杀 GTE。** Gemini 3 Pro / Flash 净评级最高，在知情少数（狼）与不知情多数（民/预/医）两侧贡献都偏正；中档出现角色短板（如部分模型当预言时负贡献——私信息难藏）；GPT-5 mini 全角色系统性偏弱。启发式 KSR / IRP / VSS 等区分度很差（共识投票抬高 VSS），反衬 GTE 的必要。成本–性能图上，棋与狼人 Pareto 形状相近，**扑克会重塑前沿并重排谁划算**——跨游戏不能合成一个「总战略分」糊弄过去。

## 🔬 最有意思的部分

1. **地面真值 vs 主观裁判是评测栈分岔，不是程度差。** Game Arena 明确站「可检验结果」一侧；和本日归档的 DIAL（去偏 + 人味校准）互补：能程序判定就别请 judge，不能判定再上 DIAL/JEV 那一套。
2. **Harness 细节就是测量定义。** 棋不给合法着列表、重试不解释原因；扑克写死 EV/GTO 目标并强制推理痕迹；狼人事件总线强制信息不对称、团队胜优先于存活。换一套 prompt/重试策略，榜会变——这和 agent harness 改 Manifest 会改行为是同一教训。
3. **样本量是扑克类评测的第一公民。** 相对 PokerBattle ~3.8k 手/模、PokerBench <2k，这里每模 ~18 万手 + duplicate；没有这个量，高方差下「谁强」只是运气故事。
4. **跨游戏分叉是产品，不是 bug。** 作者用它论证「战略」多面：规划/搜索 ≠ 信念更新/风险 ≠ 欺骗与社会推理；扩展游戏套件比追求单一总榜更有信息量。
5. **狼人杀兼作安全沙盒。** 同一模型既要演骗（狼）又要防骗（民），可在可控环境里红队欺骗能力与检测能力，而不必上真实部署。
6. **开放轨迹 + Reasoning Panel** 让评测不只报胜率：可回放「何时开始崩盘、非法着如何堆积、角色贡献从哪来」——对应 Byron 想要的失败→golden 反馈环的数据原料。

## 🤔 我的判断

定位：**以地面真值对战为核心的、可扩展 LLM 战略评测基础设施**（共享文本 harness + 三信息范式试点 + 域适配指标与大规模统计），附带开源轨迹与可视化。亮点三：

1. 把「静态饱和 / 主观 Arena 噪声」两条死路推到第三路，并真的堆出扑克级样本量与狼人 GTE；
2. 显式写出 harness 契约（重试、目标函数、可见性、方差控制）——评测结果可复现的前提；
3. 跨游戏排名分叉被当作结论而非瑕疵，逼工程别迷信单一总榜。

局限：首批只覆盖棋 / 扑克 / 狼人，离谈判、长程规划、混合动机合作等仍远；固定对局配额对碾压局浪费算力（作者自己提自适应调度）；模型上下架会撕碎纵向可比图；异构游戏的元评分仍开放；当前「战略」不等于 coding-agent / MCP 工具面能力——**不能直接当 SWE harness 的替代榜**。对 Byron：在评测栈里加一条硬门禁——**有程序可判定结果的任务，默认走环境胜负 / 单测 / 规则检查；只把开放主观面交给 LLM-as-judge，并用 DIAL 类方法做人味校准**。搭 agent 评测时，抄 Game Arena 的不是「再做一个狼人杀」，而是：统一接口、非法动作策略、方差设计、角色/切片分解、全量轨迹入库。和 [WhatWorkedBench](./20260924_WhatWorkedBench_论文解读_选对配置不等于懂干预_实验理解要交整张效应表.md) / [SchrodingerRepo](./20260924_SchrodingerRepo_论文解读_SWE榜分可能记住了仓库长相_评测时再坍缩出陌生同构仓.md) 合读：榜要防记忆与切片幻觉，Game Arena 用对手进化与 ground-truth 抗饱和，SWE 侧仍需同构仓/效应表那一套。

一句话收束：当胜负能被环境判定时，**最干净的裁判是规则本身**——Game Arena 把这句话做成了可扩展的对战基建，主观 LLM-as-judge 该退到它够不着的地方。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
