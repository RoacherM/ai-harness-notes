---
month: 202609
week: 4
date: 2026-09-22
type: 论文解读
slug: RecreationWorld
---

# 对着跑着的参考应用重写一遍：混合 CUA 才算真干活

你有没有过这种体验：GUI agent 会点不会写，终端/coding agent 会写却看不见自己产物长什么样——真实数字工作要的是**交错**，不是两条流水线首尾相接。阿里 Token Hub 这篇 RecreationWorld（arXiv:2609.22000；亦见今日 HF Daily Papers）把题面钉成 **recreation**：给你一个正在跑的参考应用，不规定工作流，agent 必须自己决定何时探索界面、何时写代码构建、何时运行并**视觉核验**自己的产物。看完我的感受是：这是 hybrid computer-use 的环境与评测基础设施——用可执行参考当 oracle，把开源应用变成可缩放、可验证的训练经验，并外推到 recreation 之外。

## 核心摘要

RecreationWorld 覆盖 **Ubuntu / macOS / Windows / Android / Web** 五平台，统一 harness 提供原生 GUI 控制与 coding 工具；候选以可构建、可启动的应用交付，隐藏测试从参考行为导出，奖励接地在执行而非源码相似。训练侧用高质量开源应用生成长程 recreation 轨迹；训出的模型在 **五个 OOD**（coding / visual coding / hybrid computer-use）基准上相对首个检查点最高可抬约 **17.9** 个百分点，并更常核验自己的渲染输出。持有评测 **RecreationBench**：**250** 题（每平台 50），程序断言 + 视觉断言覆盖多交互深度，且每条先在参考上验证、再经人工审阅后冻结。榜首 GPT-6 Astra 总体约 **58.1% / 58.06%**，但 **Prog=100%** 的应用仅 **2.8%**；分析显示 agent 更会复现静态界面结构，弱于交互与计算结果，生成应用也比参考更小、更单体。代码、环境与测试套件已开源（QwenLM/RecreationWorld · recreation-bench.cc）。

## 论文信息

- **标题**：RecreationWorld: Scalable and Verifiable Environments for Hybrid Computer-Use Agents
- **作者**：Shuai Bai、Jiayong Deng、Sicheng Fan、Yikun Fu、Chang Gao 等（Alibaba Token Hub / Alibaba Group；完整作者见论文与仓库 bib）
- **机构**：Alibaba Token Hub, Alibaba Group
- **链接**：https://arxiv.org/abs/2609.22000 （v1，2026-09-18）· 代码 https://github.com/QwenLM/RecreationWorld · 榜单站点 https://recreation-bench.cc/ · 数据 https://huggingface.co/datasets/Qwen/RecreationBench
- **观察时间**：2026-09-22（HF Daily Papers）

---

## 🎯 为何「混合」必须做成环境，而不是提示词口号

CUA 长期两条线：图形交互 vs 代码/命令行软件开发。各自的盲区对称——GUI agent 造不出界面背后的软件；终端 agent 看不见自己动作在界面上产生的后果。真实工作要求二者在同一长程环里**自主切换**：探索规格 → 实现 → 构建启动 → 对照参考做行为/视觉核验 → 再改。

Recreation 被选为纯净题型：运行中的参考同时是行为规格来源与隐藏测试的 oracle；成功标准客观（断言过多少），实现语言/框架/架构自由（实现无关监督）。任务池可随开源应用刷新而扩展，不必把人力规格写成瓶颈——这和同日 [CodeMidas](./20260922_CodeMidas_论文解读_只靠源码榨出五千可验证coding_RL环境.md)「从源码榨 RL 环境」是同一类缩放哲学，只是对象从「库内功能再实现」换成「整应用行为再实现 + GUI」。

## 🏗️ 任务环与五平台契约

每个任务给：高层次请求、对运行参考的交互访问、GUI 控制与软件开发工具。Agent 自决 explore–implement–verify 循环，无规定阶段顺序。最终候选由冻结的程序断言 + 视觉断言打分；构建/启动失败记零，否则按通过断言比例给分级信号。

平台交付与测试接口各异（AT-SPI、AXUIElement、UI Automation、UiAutomator、浏览器断言等），但能力定义同一：从运行参考恢复行为规格 → 写成代码 → 操作产物闭环。视觉断言补结构化无障碍读不到的布局/颜色/画布等。测试还按导航深度与结果特异性审计，避免退化成「看得见的控件清单」。

轨迹结构（RecreationBench）：中位约 **282.5** 次顶层 tool call，每 100 次调用约 **9.08** 次 GUI↔代码编辑切换——长程来自反馈环本身，而非人为步数上限。对照邻近基准：ProgramBench 更长但无 GUI；OSWorld 2.0 侧重量计算机工具调用——recreation 卡在「又长又要切换」的中间带。

## 🧪 RecreationBench：58% 均值，2.8% 全过程序满分

十模型主结果（平台等权；Average = Prog 与 VLM 均值；仓库 README 表与论文 Table 3 一致）：

| 模型 | Prog (%) | VLM (%) | Average (%) | Prog≥90% | Prog=100% |
| --- | ---: | ---: | ---: | ---: | ---: |
| GPT-6 Astra | 58.19 | 57.92 | 58.06 | 17.60 | **2.80** |
| Claude Opus 5 | 45.99 | 42.34 | 44.16 | 5.53 | 0.80 |
| GPT-5.6 Sol | 40.63 | 43.49 | 42.06 | 5.20 | 0.40 |
| Grok 4.6 | 39.02 | 34.45 | 36.73 | 1.60 | 0.00 |
| Qwen3.8-Max-0902 | 35.53 | 34.07 | 34.80 | 2.00 | 0.00 |
| … | … | … | … | … | … |

榜首也几乎做不到「程序断言全过」——hybrid CUA 的硬缺口暴露在交互深度与计算输出，而不是截一张静态壳。训练迁移：共享高分轨迹训两个初始化，五条 OOD 均高于首检点，增益最高约 **+17.9** pp；并观察到更频繁的自渲染核验——recreation 监督不只刷榜内分数，还带回可迁移的混合行为。

另有可编程交互运行时消融（持久 Node REPL / 组合 GUI SDK vs 每次原语就归还控制的 direct-MCP）：主榜分数仍以 standard direct-MCP 接口为准，该运行时实验单独报告，不并入 Table 3。

## 🔬 最有意思的部分：参考即裁判，实现无关

把「可执行参考 → 隐藏断言」做成评测层，比「LLM 看截图打分」硬一个数量级：断言先在参考上验证、再人工审、再冻结——这是 verifier 工程，不是临时 judge prompt。和 [LayerIsolatedEval](./20260922_LayerIsolatedEval_论文解读_总分几乎不动切片却崩了.md) 同构：aggregate 58% 掩盖「全套 Prog 仅 2.8%」；该切片报、不该只报均值。与 [CodeMidas](./20260922_CodeMidas_论文解读_只靠源码榨出五千可验证coding_RL环境.md) 对照：两者都坚持执行接地奖励；RecreationWorld 额外强迫 GUI↔代码切换与视觉自检，补上 coding-only RL 环境缺的「看见自己产物」一环。

## 🤔 我的判断

定位：**hybrid CUA 的可缩放环境 + 持有基准 + 迁移证据**，方法与基础设施各半。亮点三：

1. recreation 题型同时解决「规格从哪来」与「分怎么打」——运行参考一身二任；
2. 五平台原生语义保留 + 统一 harness，评测契约清楚（250=5×50）；
3. OOD 迁移与自核验频率上升，说明训的是混合能力而非刷 recreation 皮。

局限：前沿模型成本极高（README 估 GPT-6 Astra 约 115 USD/task 量级，含缓存假设）；全过率极低，短线更适合当诊断床而非唯一人模 KPI；生成应用偏小偏单体，距生产级复现仍远；训练轨迹规模与过滤细节需读附录才能复现飞轮。对 Byron：若 harness/MCP 要吃「真电脑工作」，可把 RecreationWorld 式环接进外环——**(a)** 参考应用 → 隐藏程序/视觉断言 → explore–implement–verify；**(b)** 指标固定拆 Prog/VLM、深度切片与 =100% 尾部，勿只报平均；**(c)** MCP 工具层区分「原子 GUI」与「可持久编程运行时」，分别测开销与质量；**(d)** 与 CodeMidas 环境、CategoryAware 式分布感知后训练串联：先有可验证混合课表，再谈类别专家与蒸馏。

一句话收束：能点会写还不够——对着一个还在跑的参考，把行为重做出来并自己验过，才是 hybrid computer-use 该考的那门课。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
