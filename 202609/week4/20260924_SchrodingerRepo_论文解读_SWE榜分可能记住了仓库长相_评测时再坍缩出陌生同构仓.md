---
month: 202609
week: 4
date: 2026-09-24
type: 论文解读
slug: SchrodingerRepo
---

# SWE 榜分可能记住了仓库长相：评测时再「坍缩」出陌生同构仓

你有没有过这种体验：SWE-bench Verified 又刷高了，却心里发虚——Django 路径、`get_FOO_display`、熟悉的文件布局，到底是「真会修」还是「训练里见过这副脸」？上交 / 西交 / 华东师大这篇 SchrodingerRepo（Schrödinger's Code Repository，arXiv:2609.27891）把测试仓库当成 **评测时才坍缩的潜变量**：用种子条件变换擦掉命名、路径、布局与实现长相，却保住可执行行为与测例语义。看完我的感受是：coding-agent 排行榜可能在测 **memorized Django/path 线索**，不全是 harness 鲁棒性——评测必须在运行时实例化仓库视图；这和 [ImpossibleRubrics](./20260923_ImpossibleRubrics_论文解读_定制Rubric教攻击者怎么造假_忠实证书才是0Exploit.md)「分数可被表面结构骗」同一条怀疑链，只是从 rubric 挪到了 **repo 表象**。

## 核心摘要

SchrodingerRepo 不把 SWE 实例钉死成一份公开 canonical 仓库快照，而是在 agent 进入评测环境时，按种子生成语义等价、行为可执行、却「长得陌生」的仓库视图。四级变换：(1) 问题陈述释义 + 校验；(2) **可逆命名空间重映射**（路径 / 符号；主杀伤）；(3) 依赖安全拓扑打乱的文件内定义重排；(4) 对 golden-patch 相关区域做功能保持改写（保留标准要求 `Pass_to_Pass=1`、`Fail_to_Pass=0`）。Agent 脚手架用 mini-swe-agent；模型含 GPT-5.1、GPT-5.4-mini、DeepSeek-v4-Flash、Gemini-3.1-Flash-Lite。SWE-bench Verified 全变换 Pass@1：GPT-5.1 **44.6→36.2（−8.4）**，GPT-5.4-mini **46.8→35.6（−11.2）**，DeepSeek-v4-Flash **72.8→66.8（−6.0）**，Gemini **56.7→42.3（−14.4）**；输入 token 常 **>2.5×**。额外动作里 **81.6–83.6%** 是探索（navigate/search/read/probe），不是 edit/test。人类泄漏探针：`>65%` 实例有泄漏证据，`>18%` 可到 patch/test 级回想。RQ4 用 cutoff 后的 SWE-rebench（2026-03）：Pass@1 不变 **17.27%**，但动作 / token 仍升——变换主要剥熟悉度，不是把题变难。SWE-QA 同样掉分、抬成本。

## 论文信息

- **标题**：Schrödinger's Code Repository: Have LLMs Learned SWE-bench or Memorized It?
- **作者**：Silin Chen*、Yufei Yang*、Xiaodong Gu†、Yuling Shi、Chengcheng Wan、Haibing Guan（*同等贡献；†通讯）
- **机构**：上海交通大学 · 西安交通大学 · 华东师范大学 / 上海创智学院
- **链接**：https://arxiv.org/abs/2609.27891 · 代码 https://github.com/cslsolow/Schrodinger-Repo
- **观察时间**：2026-09-24（HF Daily Papers）

---

## 🎯 为什么这件事值得写

SWE-bench 系把真实 GitHub issue 做成可执行修补任务，却几乎总是挂在 **同一份公开仓库长相** 上：路径、符号、Django 惯用名、文件布局，训练语料里反复出现。Live / rebench / Pro 用时间新鲜度减污染，但仍受规模与题型覆盖限制。OpenAI 已公开质疑 Verified 是否还能量到前沿能力；本文先做人类泄漏探针，再系统改 **仓库表象**，而不是只换一批新 issue。

对 Byron（coding-agent / harness / evals）：排行榜若部分测的是「认出 django/db/models」而不是「在陌生同构仓里定位与修补」，那 Meta-Harness、Harness-Zero、CategoryAwareSWE 一类工作的外推都要打折——**评测契约本身要变成运行时实例化**。

## 🏗️ 机制：评测时潜变量 + 四级同构变换

核心主张：仓库表示不应是冻结的基准工件，而应是 **evaluation-time latent**——每次 agent 进环境才按种子坍缩成具体视图。映射可逆：观测层翻译成虚拟命名空间，动作再翻回真实 SWE 环境执行；最终 patch 从仓库状态 diff 回原坐标再打分。

| Level | 做什么 | 意图 |
| --- | --- | --- |
| 1 问题陈述重建 | 释义 / 重排 / 去冗余，再经 LLM verifier 校验语义 | 擦掉 canonical 措辞 |
| 2 命名空间映射 | AST 抽仓库内标识符 → 子词级种子映射（保 CamelCase/snake_case 等）；路径 / 模块一并虚化；外库 API 不动 | **擦掉最强熟悉线索** |
| 3 文件内布局重排 | 可重排定义段建依赖图，随机合法拓扑序 | 擦掉「定义总在那几行」的先验 |
| 4 功能保持改写 | 只动 golden-patch 相关区域；离线改写 + 测例闸门 `Pass_to_Pass=1`、`Fail_to_Pass=0` | 擦掉 fix 附近实现长相 |

L3/L4 为控成本只覆盖 golden-patch 相关文件 / 区域；每级每实例用三粒种子，结果平均。有效性：人工抽检 100 例同题；L3/L4 在官方执行环境跑测例闸门。

## 🧪 关键证据：掉分、涨成本、主杀伤是 Level 2

**SWE-bench Verified（Table I，Pass@1）**

| 模型 | Baseline | Full SchrodingerRepo | Δ |
| --- | --- | --- | --- |
| GPT-5.1 | 44.6% | 36.2% | **−8.4** |
| GPT-5.4-mini | 46.8% | 35.6% | **−11.2** |
| DeepSeek-v4-Flash | 72.8% | 66.8% | **−6.0** |
| Gemini-3.1-Flash-Lite | 56.7% | 42.3% | **−14.4** |

级联拆解：Level 1 几乎不动 Pass@1；**Level 2 单独**就把 GPT-5.1 / 5.4-mini / DeepSeek 打到 −7.4 / −6.4 / −6.0，并拉高动作与输入 token（如 GPT-5.1 输入 **+162%**）。L3/L4 掉分较轻（约 0.8–3.4 pp），但仍抬交互成本。全开仍最惨——熟悉度分布在多层表象，不只命名。

**行为（RQ2）**：相对 baseline 多出来的动作，DeepSeek **83.6%**、GPT-5.4-mini **81.6%** 落在 navigate/search/read/probe；edit/test 只占约一成半。DeepSeek 更爱 probe，Pass@1 相对保住更多但更贵；GPT-5.4-mini 更「省探索」，掉分更狠——**探索强度 ↔ 表象鲁棒性** 的权衡。

**SWE-QA（RQ3）**：GPT-5.4-mini 均分 **70.35→65.71（−4.64）**，动作 **+18.15%**；DeepSeek 分几乎不动（72.97→72.42）但动作 **+43.02%**、输入 token **+59%**——同样是「多花钱重建上下文」。

**RQ4（SWE-rebench 2026-03，cutoff 后）**：GPT-5.4-mini Pass@1 钉死 **17.27%**，动作 **15.82→17.11（+8.15%）**，输入 token **+22%**。结论：变换不是把题变难，而是剥掉熟悉线索。

**动机实验**：人类按语义单元渐进揭示 issue；在看不到仓库 / 金标 patch 时，各模型 **>65%** 实例有泄漏证据，**>18%** 可到 patch/test 级回想——canonical 榜分从一开始就混进了记忆。

案例 django__django-11999：原视图靠 `get_.*_display` 很快钉到 `fields/__init__.py`；变换后路径变成 `storage_engine/object_models/...`，动作从约 **37→217**，最终仍交同构修补——过程变了，题没变。

## 🔬 最有意思的部分

1. **评测对象从「固定仓库快照」换成「同构视图分布」**：seed-conditioned、可逆、可复跑——比一次性扰动基准更难被「背答案」。
2. **主杀伤是命名空间，不是题面措辞**：Level 1 几乎白做；榜分熟悉度主要在 **repo-owned names / paths**。
3. **成本暴涨主要是探索债**：>80% 额外动作在 localization——harness 若只报 Pass@1，会掩盖「其实在慌张翻仓」。
4. **cutoff 对照把「变难」假说否了**：Pass@1 不变、成本仍升——这是熟悉度税，不是难度税。
5. **与静态扰动线分野**：PoorCodeSumEval / RepoMirage 等改完仍是固定工件；SchrodingerRepo 强调 **每次评测再实例化**，降低被再吸进训练集的风险。

## 🤔 我的判断

定位：**仓库级 coding-agent 评测的表象鲁棒性基础设施**，不是新修 bug agent。亮点三：

1. 把「学到了还是背到了」操作成可复现的四级同构变换 + 可逆执行；
2. Verified 上 6–14 pp 掉分 + >2.5× token + 探索占比数字够硬；
3. RQ4 用时间外集把「只是变难」拆开。

局限：主战场仍是 Python / CLI agent；L3/L4 只盖 golden 区域，低估全仓重排难度；Gemini 子集是强泄漏 300 例，不全量；IDE / LSP 工具链适配未完成。对 Byron（MMP / mini-swe 类脚手架、排行榜读数、evals 闭环）：**默认别把 Verified 总分当 harness 鲁棒性**；若自建评测，至少加一层 seed 命名空间映射，并切片报探索动作占比与 token——和 [LayerIsolatedEval](./20260922_LayerIsolatedEval_论文解读_总分几乎不动切片却崩了.md) 一样：**总分会撒谎，切片与过程指标才暴露机制**。旁注：训练侧 [CodeMidas](./20260922_CodeMidas_论文解读_只靠源码榨出五千可验证coding_RL环境.md) 从源码榨环境；评测侧 SchrodingerRepo 要求环境视图在评测时再坍缩——飞轮两端都要防「背仓库脸」。

一句话收束：SWE 榜分要配一句免责声明——**在没把仓库长相打乱之前，你可能只是在测记忆，不是在测工程推理**。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
