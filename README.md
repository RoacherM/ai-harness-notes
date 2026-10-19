# ai-harness-notes

面向 **AI coding-agent / skills / harness / MCP** 的中文深度归档。

文风对齐 [shibing624/ai-paper-analysis](https://github.com/shibing624/ai-paper-analysis)：钩子标题 → 体验开场 → 核心摘要 → 信息卡 → 带 emoji 的展开章节 →「我的判断」收束。写机制与判断，不是链接清单或 README 翻译。

维护：[@RoacherM](https://github.com/RoacherM) · 由助手 BB 协助撰写与推送。

## 目录约定（月 / 周内筛选）

```text
YYYYMM/
  week1/    # 当月 1–7 日
  week2/    # 8–14 日
  week3/    # 15–21 日
  week4/    # 22–28 日
  week5/    # 29–31 日（仅有才建）
    YYYYMMDD_简称_论文解读_中文标题.md
    YYYYMMDD_简称_项目解读_中文标题.md
```

每篇文首 YAML：

```yaml
---
month: 202609
week: 3
date: 2026-09-16
type: 项目解读
slug: RSI-Harness
---
```

## 索引

| 月 | 周 | 篇目 |
| --- | --- | --- |
| 202610 | [week3](./202610/week3/) | [DeRef](./202610/week3/20261015_DeRef_论文解读_裁判看见参考才认对_推导缺口卡死工具参数验收.md) · [ARBITER](./202610/week3/20261015_ARBITER_论文解读_官方判对却不说错哪_可验证反例当代码裁判证据.md) · [EMHO](./202610/week3/20261015_EMHO_论文解读_模型冻住改脚手架_经验轨迹让具身harness自己进化.md) · [ActWrong](./202610/week3/20261015_ActWrong_论文解读_读对却做错_System_One在agent循环里的决策失败.md) · [FailConf](./202610/week3/20261015_FailConf_论文解读_agent能学会预言自己失败吗_结果锚定置信度校准.md) · [WorkForge](./202610/week3/20261015_WorkForge_论文解读_可验证工作环境也能规模化_真实工作区事实锚定任务与裁判.md) · [JevAgentHarness](./202610/week3/20261016_JevAgentHarness_论文解读_孤立题答对不等于能控住agent_决策模型要拟合阈值才共享控制.md) · [Doc2Harness](./202610/week3/20261016_Doc2Harness_论文解读_下游任务榜测不了脚手架工程_从文档造完整harness才见真章.md) · [Cortex](./202610/week3/20261016_Cortex_论文解读_加一条skill别靠一次成功_测量门控才留下rules.md) · [SWE-at-Scale](./202610/week3/20261016_SWE-at-Scale_论文解读_有限真实修复怎么造出无限难考场_把修bug翻成造bug监督.md) · [Wedge](./202610/week3/20261016_Wedge_论文解读_同一任务三种死法_弱点导向造环境才喂得进持续学习.md) · [SelfEvolvingHarness](./202610/week3/20261019_SelfEvolvingHarness_论文解读_改脚手架不能只看这次涨分_新鲜验证集加回归预算才配叫自进化.md) · [VerifierHardening](./202610/week3/20261019_VerifierHardening_论文解读_今天的弱模型给明天的强模型修考场_黑客修补循环把作弊率压下来.md) · [CarryOn](./202610/week3/20261019_CarryOn_论文解读_别再压缩对话历史_让文件和状态本身当进度载体.md) · [Harness2](./202610/week3/20261019_Harness2_论文解读_没有标准答案也能改脚手架_对比探针加改进库让下一次改得更好.md) · [Mosaic](./202610/week3/20261019_Mosaic_论文解读_压缩和子代理把轨迹切成多段_RL别再把同一个优势广播给每一段.md) · [LEASH](./202610/week3/20261019_LEASH_论文解读_小模型先干看隐状态像不像失败_再叫旗舰接手还要拴着它.md) |
| 202610 | [week2](./202610/week2/) | [AgentGarten](./202610/week2/20261009_AgentGarten_论文解读_四轮写playbook顶过百万局RL_代码世界加神经渲染器让agent边玩边进化.md) · [Learn2Play](./202610/week2/20261009_Learn2Play_论文解读_经验别急着蒸馏成规则_原始轨迹加好harness才学得会陌生规则.md) · [EmbodiedCLBench](./202610/week2/20261012_EmbodiedCLBench_论文解读_做过一道基础题就涨三倍_再做五道却不再涨_具身持续学习卡在经验提取.md) · [SoftwareGym](./202610/week2/20261012_SoftwareGym_论文解读_一个OSWorld应用都没训_合成182个桌面软件让9B模型涨15点.md) · [ReHarness](./202610/week2/20261012_ReHarness_论文解读_反思写进记忆就不再改_反向修订让harness状态也能吃到下游反馈.md) · [Work2Skill](./202610/week2/20261012_Work2Skill_论文解读_录一遍人类操作就能写成SKILL.md_按需翻证据比流水线解析更好用.md) · [LatentGRM](./202610/week2/20261013_LatentGRM_论文解读_裁判别再逐字写评语_准则塞进潜空间压九倍轨迹仍打平显式SFT.md) · [SWE-Duel](./202610/week2/20261013_SWE-Duel_论文解读_静态榜会漏进训练又被刷穿_让agent互出题互修难度跟着作者一起涨.md) · [ProgramGym](./202610/week2/20261013_ProgramGym_论文解读_从零复现程序缺可验证考场_1648黑盒环境加角色隔离把27B抬过旗舰脚手架.md) · [ExpActivator](./202610/week2/20261013_ExpActivator_论文解读_相关不等于适用_检索经验第一步就花光_每屏重新激活才让个人GUI助理记得住.md) · [PANINI-DR](./202610/week2/20261014_PANINI-DR_论文解读_深研别把协调和研究搅在一条链里_共享工作区加验证闸让难题答对近翻倍.md) · [LongHarness](./202610/week2/20261014_LongHarness_论文解读_证据和结论别埋在对话历史里_共进化状态才把一亿token语料问对.md) · [AgentSWE](./202610/week2/20261014_AgentSWE_论文解读_coding_agent交出来看着齐其实空_冻结提交加holdout_rubric才测得出真交付.md) · [TerminalHorizon](./202610/week2/20261014_TerminalHorizon_论文解读_短轨迹只会敲命令_挑战重建加渐进扩地平线才攒出长程终端智能.md) · [SWE-TaskFlow](./202610/week2/20261014_SWE-TaskFlow_论文解读_没有放之四海的真实交互_榜该对齐目标用户的Task_Flow.md) · [COAP](./202610/week2/20261014_COAP_论文解读_具身世界当图灵机_状态写进代码决策也写代码_测试时不再调模型.md) |
| 202610 | [week1](./202610/week1/) | [Raven](./202610/week1/20261001_Raven_论文解读_别再手搓更强harness_Harness的Harness才把异构agent当可组合单元.md) · [HybridCUA](./202610/week1/20261001_HybridCUA_论文解读_光给shell不够_学会何时用CLI才不在OSWorld掉分.md) · [DerivAudit](./202610/week1/20261001_DerivAudit_论文解读_记忆不是摘抄是推导_分布式证据悖论卡死写入闸.md) · [LibraryDesignBench](./202610/week1/20261001_LibraryDesignBench_论文解读_库写对了不算_下游agent写出来才算_为人API不等于为agent.md) · [Omni-IO-Skills](./202610/week1/20261001_Omni-IO-Skills_论文解读_别改host推理核_用Skills加MCP加资产注册表把agent变omni原生.md) · [EpiCon](./202610/week1/20261001_EpiCon_论文解读_经验别锁死在一个harness_共享多模态记忆才能集体学.md) · [MidHarness](./202610/week1/20261002_MidHarness_论文解读_别再只扩整条轨迹_动作边界采样验证才救终端agent.md) · [MetaSkill](./202610/week1/20261002_MetaSkill_论文解读_别把技能塞给Target_教Builder学元技能才配得动harness.md) · [Skill-Use](./202610/week1/20261002_Skill-Use_论文解读_技能挂上了不等于会用_Trigger合规边界才测真使用.md) · [RASO](./202610/week1/20261002_RASO_论文解读_公开技能库别直接粘_跨Harness适配才把别人的skill编译进自己的.md) · [FloWright](./202610/week1/20261002_FloWright_论文解读_别只训工作流生成器_把workflow当harness才能共进化角色.md) · [GraphForge](./202610/week1/20261002_GraphForge_论文解读_真文件证据图才同时钉死任务和Rubric.md) |
| 202609 | [week3](./202609/week3/) | [RSI-Harness](./202609/week3/20260916_RSI-Harness_项目解读_切场景就是切Genome_Pi脚手架的可版本化.md) · [ComposeCL](./202609/week3/20260916_ComposeCL_论文解读_持续学习机制组合起来才扛得住百任务记忆.md) · [LLM-as-Judge](./202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) · [Agora](./202609/week3/20260918_Agora_论文解读_加agent不等于加发现_Git_DAG当集体科研共享记忆.md) · [MoModels](./202609/week3/20260918_MoModels_论文解读_模型池越大越强_异质MAS里多经常变成更差.md) · [Meta-Harness](./202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) · [Harness-Design](./202609/week3/20260918_Harness-Design_论文解读_脚手架不是一整块_组件要按模型和预算配.md) · [SoL-Pi](./202609/week3/20260918_SoL-Pi_论文解读_先省token再扩RSI_四个机制把Pi流量砍半.md) · [EvoSkill-GUI](./202609/week3/20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) · [EvoHarnessBench](./202609/week3/20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) · [Subagents-vs-Skills](./202609/week3/20260921_Subagents-vs-Skills_论文解读_技能该塞进主上下文还是开子代理.md) · [Code2Skill](./202609/week3/20260921_Code2Skill_论文解读_还没跑轨迹也能攒技能库_从两万仓库榨出百万条.md) · [GraphSkillEvo](./202609/week3/20260921_GraphSkillEvo_论文解读_技能别再写成一长串checklist_做成步骤图再进化.md) · [TrustReviewer](./202609/week3/20260921_TrustReviewer_论文解读_AI审AI审出来的裁判会塌缩_评分挤扁语义同质.md) |
| 202609 | [week4](./202609/week4/) | [RuVerBench](./202609/week4/20260922_RuVerBench_论文解读_裁判验长轨迹Rubric仍有噪声.md) · [LayerIsolatedEval](./202609/week4/20260922_LayerIsolatedEval_论文解读_总分几乎不动切片却崩了.md) · [CodeMidas](./202609/week4/20260922_CodeMidas_论文解读_只靠源码榨出五千可验证coding_RL环境.md) · [CategoryAwareSWE](./202609/week4/20260922_CategoryAwareSWE_论文解读_总分涨着一类却在掉_按类别训专家再蒸馏回来.md) · [RecreationWorld](./202609/week4/20260922_RecreationWorld_论文解读_对着跑着的参考应用重写一遍_混合CUA才算真干活.md) · [EvoOntology](./202609/week4/20260922_EvoOntology_论文解读_语义别塞进提示词_做成可进化的MCP本体层.md) · [ImpossibleRubrics](./202609/week4/20260923_ImpossibleRubrics_论文解读_定制Rubric教攻击者怎么造假_忠实证书才是0Exploit.md) · [RRSI](./202609/week4/20260923_RRSI_论文解读_Harness_RSI会过拟合正则提案与筛选才保住OOD.md) · [Harness-Zero](./202609/week4/20260923_Harness-Zero_论文解读_把专用Harness行为烤进权重_拆掉仍能超过挂着它的基线.md) · [AIDE2-RSI](./202609/week4/20260923_AIDE2-RSI_论文解读_双环RSI改自己代码_8天刷出能打两年人工工程的研究Agent.md) · [SkillSpec](./202609/week4/20260923_SkillSpec_论文解读_第一次用Hoare规格验Agent技能_515个近半有缺陷.md) · [ACLArena](./202609/week4/20260923_ACLArena_论文解读_Agent多阶段后训练会忘_共享权重合并有取舍_LoRA专家路由更近Oracle.md) · [LIMBO](./202609/week4/20260924_LIMBO_论文解读_经验回放别全塞进提示_按任务在线配记忆预算.md) · [Paper2Agent](./202609/week4/20260924_Paper2Agent_论文解读_论文别只当PDF_锁死验证过的MCP工具比自由写代码更可靠.md) · [JEV-as-a-Judge](./202609/week4/20260924_JEV-as-a-Judge_论文解读_自信就收下不确定就升级_决策裁判三分钱顶住强模型.md) · [SchrodingerRepo](./202609/week4/20260924_SchrodingerRepo_论文解读_SWE榜分可能记住了仓库长相_评测时再坍缩出陌生同构仓.md) · [WhatWorkedBench](./202609/week4/20260924_WhatWorkedBench_论文解读_选对配置不等于懂干预_实验理解要交整张效应表.md) · [VHD-Play](./202609/week4/20260924_VHD-Play_论文解读_先解数学模型再包成有状态工具_三千环境几分钱一份.md) · [AEWM](./202609/week4/20260925_AEWM_论文解读_别再猜工具回包_判动作改状态才是Agent世界模型.md) · [SkillPivot](./202609/week4/20260925_SkillPivot_论文解读_失败别整轨反思_先找偏航点再对比后缀改技能.md) · [ExactlyOnce](./202609/week4/20260925_ExactlyOnce_论文解读_恰好一次住在工具合同里_Harness几乎不决定.md) · [Qwen-Planner-Agent](./202609/week4/20260925_Qwen-Planner-Agent_论文解读_模型练完还不够_Skills_Memory_Harness要和权重一起共进化.md) · [JITMem](./202609/week4/20260925_JITMem_论文解读_记忆别在写入时裁剪_读时按任务现场策展同一条轨迹能榨出不同课.md) · [DIAL](./202609/week4/20260928_DIAL_论文解读_消掉位置偏差还不够_少量人工把多裁判偏好校准成人味.md) · [MutableTranscripts](./202609/week4/20260928_MutableTranscripts_论文解读_对话别只追加纠错_把历史改成可编辑状态剪掉上下文污染.md) · [JustAskJev](./202609/week4/20260928_JustAskJev_论文解读_对齐失败检测别为每条准则再解码_一次问多题零样本就能筛.md) · [GameArena](./202609/week4/20260928_GameArena_论文解读_主观裁判太吵静态榜又饱和_用胜负当地面真值的对战评测台.md) |
| 202609 | [week5](./202609/week5/) | [TimeEvo](./202609/week5/20260929_TimeEvo_论文解读_专家工具库帮了也害了_失败驱动自进化才敢上工具.md) · [SLCA-GRPO](./202609/week5/20260929_SLCA-GRPO_论文解读_工具段和总结段别共用一个优势_分段锁信用.md) · [DSSR](./202609/week5/20260929_DSSR_论文解读_预算够了仍忘_写时后悔才是常驻记忆瓶颈.md) · [EMem-Bench](./202609/week5/20260929_EMem-Bench_论文解读_长程具身记忆四缺_外部记忆把SR抬二十多点.md) · [AdaTutoRank](./202609/week5/20260929_AdaTutoRank_论文解读_集合级标量奖励太稀_自适应辅导才把信用压到token.md) · [DRM](./202609/week5/20260929_DRM_论文解读_奖励别再点估计_扩散头才装得下多峰偏好.md) · [Gagar](./202609/week5/20260930_Gagar_论文解读_测试过了还不够_组内agentic裁判才把信用拨给可合并补丁.md) · [TraceDance](./202609/week5/20260930_TraceDance_论文解读_部署痕迹变成行为基准_决策点续写才测过程.md) · [ExpVoyager](./202609/week5/20260930_ExpVoyager_论文解读_技能别提前烤死_按需在经验上导航再合成.md) · [WideSWE](./202609/week5/20260930_WideSWE_论文解读_单仓绿了不算完_跨仓协调才是coding_agent真考场.md) · [Skill2Env](./202609/week5/20260930_Skill2Env_论文解读_技能不够成考场_能力难度模式才把环境造硬.md) · [QwenGyre](./202609/week5/20260930_QwenGyre_论文解读_xLong黑盒harness上RL_弹性调度加轨迹树才吃得下.md) |

## 单篇结构（见 TEMPLATE.md）

1. 钩子标题 + 「你有没有过这种体验」开场 + 看完感受  
2. `## 核心摘要`  
3. `## 论文信息` / `## 项目信息`  
4. `---` 后用 `🎯🏗️🧪📈🔬🤔` 等章节展开  
5. 以 `## 🤔 我的判断` 收束；文末固定互动 footer  

## License

笔记以 CC BY 4.0 共享；引用论文与项目请遵循其原许可证。
