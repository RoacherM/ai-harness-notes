---
month: 202611
week: 1
date: 2026-11-06
type: 论文解读
slug: Harness-S
---

# 别再找「万能 harness」：失败时才调兄弟脚手架，学到的行为却能留回底座。Harness-S 让 Qwen3.5-4B 在 AppWorld Pass@3 从 GRPO 的 60.31 提到 67.56，仍在原 H0 上跑

你有没有过这种体验：同一套底座脚手架 H0，你顺着失败轨迹改出一串变体。每个变体都修好了一小撮题，却把原先会做的题弄坏；几个变体拼起来的 oracle 覆盖远高于任一个，也高于把 H0 多采样到同样次数。更糟的是，政策一学，原先「好用」的变体排名会翻盘。你要是把补丁全并进一个「最终 harness」，往往两头不讨好；要是让 harness 和政策一起缠着演化，成本又贵，而且换回 H0 时增益可能蒸发。

这就是 harness 演化里作者点出的三件事实：**Locality（局部修好、局部回退）、Complementarity（互补）、Nonstationarity（随政策漂移）**。核心问题不是「哪一个 harness 永远最强」，而是：**能不能只在底座全挂时借用兄弟脚手架救命，却把救回来的行为烤进同一条政策，最后仍在原 H0 上变强？**

上海交大与小红书等机构的 **Harness-S** 给出一套拆开的配方：先离线长出固定的 Sibling Harness Pool，再在训练里做 Complementary Harness Rescue，外加会随政策刷新的 Adaptive Harness Router。看完我的感受是：**这是把「专用执行支持」当成可回收的探索信号，而不是部署时必须挂着的外挂。**

## 核心摘要

Harness-S 不追求一个压过所有兄弟的万能 harness。它先做 Directional Harness Evolution：从共同底座 H0 出发，按根因方向分支改 harness，用联合验证覆盖挑父代，保留所有可执行、协议合规的子代，得到训练期冻结的 Sibling Harness Pool。政策训练用 GRPO：每个任务先在 H0 下打一组 rollout；只有整组全失败时，Adaptive Harness Router 才按 Harness Behavior Profiles 与多 LLM 投票选 Top-k 兄弟，让同一政策在任务初始状态上重跑，用这组救援轨迹替换失败组再更新。Profile 会随政策学习做 policy-aware re-anchoring，对抗非平稳。评测在 AppWorld、SWE-Gym、DBBench 上对 Qwen3.5-4B/9B 报 Pass@3 与 Avg@3（三次独立随机跑、每任务三次执行）。完整 Harness-S 在仍测 H0 时，4B AppWorld Pass@3 到 67.56±1.87（GRPO 60.31，H0 rescue 61.83）；SWE-Gym Pass@3 到 38.75±2.22（GRPO 24.38）；DBBench Pass@3 到 73.33±2.36（GRPO 66.67）。同一检查点切到任务级 Hrank1 还能再涨一截。行为标注与 holdout 案例显示分页、先读 schema、测试阻塞后换断言等模式在回到 H0 后仍更常见。代码：https://github.com/DeepExperience/Harness-S。

## 论文信息

- **标题**：Harness-S: Learning from Sibling Harnesses for Generalizable Agent Training
- **作者**：Kangning Zhang、Wenxiang Jiao（通讯）、Bowen Fang、Shuai Shao、Jiachen Zhu、Rong Shan、Shijian Wang、Hao Wang、Yuan Lu（通讯）、Weiwen Liu（通讯）、Weinan Zhang（通讯）
- **机构**：上海交通大学；小红书；中关村人工智能研究院；东南大学；Carnegie Mellon University
- **链接**：https://arxiv.org/abs/2611.04277 （2026-11-03 提交；HF Daily Papers 2026-11-06）
- **代码**：https://github.com/DeepExperience/Harness-S

---

## 🎯 为什么这件事值得写

近期 Meta-Harness、HarnessX、Life-Harness 一类工作，都在「同一执行系统里从 H0 长出变体」。作者用 Qwen3.5-4B 在 AppWorld 上把三件经验事实钉死：

1. **Locality**：相对 H0，各 sibling 修好与回退的任务集合不同，净增益有限。
2. **Complementarity**：各取一次 rollout 的 oracle union，覆盖高于任一 Hi，也高于同预算的 H0 pass@K。
3. **Nonstationarity**：固定 harness 相对 H0 的优势会随政策检查点翻转，甚至变负。

于是两条常见路都别扭：硬合成一个万能 harness，容易丢掉互补专长；harness–政策耦合演化又贵、又难复用，回到 H0 时未必还在。Harness-S 的野心是：**把兄弟 harness 当训练期探索器，目标政策部署时仍可只挂 H0。**

## 🏗️ 机制：离线池 + 全失败才救援

### Directional Harness Evolution

从 H0 诊断失败根因，每个子 harness 只打一小簇相关原因；用联合验证覆盖选下一轮父代；所有能跑通协议的子代都进池，训练期不再改池。图里举例：有的逼你先读 schema，有的逼你翻页翻到底，有的收紧过滤条件。

### Complementary Harness Rescue Training

共享政策 πθ。每个训练任务先在 H0 下采一组（文中常用 8 条）。

- 组内已有成功：保留，按常规 GRPO 更新。
- **全部失败**：Router 选 Top-k 辅助 harness，同一政策从任务初始状态各采一条；**整组替换**失败组再进 GRPO。

目标不是部署时永远挂着这些兄弟，而是让政策在互补执行支持下产生可强化的对比信号，再把行为留回 H0。

### Adaptive Harness Router

Harness Behavior Profiles 描述「相对 H0，当前政策在什么任务条件下被该变体帮到或拖累」。多 LLM 独立投票，再结合验证集相似任务成功率。训练过程中周期性用当前政策轨迹做 profile re-anchoring，对抗非平稳。推理期也可复用 Router 做 Rank-1（每任务先选一个 harness 再跑），在不增加 executor rollout 数时再挤一点分。

## 🧪 关键证据

主表（非前沿方法内加粗最优；Harness-S 为三次独立跑的 mean±std）：

| 设置 (4B, 测 H0) | AppWorld Pass@3 | SWE-Gym Pass@3 | DBBench Pass@3 |
| --- | --- | --- | --- |
| Base | 42.62 | 19.38 | 59.00 |
| GRPO | 60.31 | 24.38 | 66.67 |
| GRPO (H0 rescue) | 61.83 | 25.63 | 67.33 |
| Harness-S | **67.56±1.87** | **38.75±2.22** | **73.33±2.36** |

同检查点改测 Hrank1：4B AppWorld Pass@3 到 68.32，SWE-Gym 到 42.50，DBBench 到 75.00；9B Hrank1 更高（AppWorld 75.19）。固定混合（H0+Claude Code 或 H0+Top-4）反而接近甚至低于 plain GRPO，说明「训练时乱挂一堆 harness」不等于会迁移。只做 harness evolution、冻住起始政策的 Harness-S (HE only) 明显弱于完整版，说明增益主要来自政策侧吸收。

训练过程上，Harness-S 有效组占比与含成功救援的份额全程领先；AppWorld 4B 全程有效组率约 79.38%，对照 H0 rescue 70.94%、GRPO 64.69%。行为频率表：回到 H0 后，Harness-S 更常先查 API schema、更多分页与多调用闭环；SWE-Gym 上测试阻塞后切断言约 48.5% vs GRPO 13.3%。Holdout 案例里，集合交集约束与依赖邮箱验证码的找回流程能在 H0 上复现，而 GRPO 仍失败。

## 🔬 最有意思的部分

**救援触发条件极苛刻：只有 H0 全灭才换组。** 这避免了「明明底座会做还去学外挂捷径」。固定混合接近 GRPO，则说明关键不是「多看几种 harness 轨迹」本身，而是 **按任务、在真正没信号时，选互补兄弟来制造组内对比**。Router 消融里 Top-8 Adaptive 最终测试最强；随机抽池或无 profile 的规则路由也有用，但弱一截。Rank-1 在相同 executor 次数下的额外增益，则是「训练期学会的路由」还能当推理插件，但论文主叙事仍强调 H0 上的政策迁移。

## 🤔 我的判断

定位：**测量清楚的 harness 互补性之后，给「跨 harness 探索 → 单 harness 部署」的一套可复现训练配方。**

亮点三条：

1. 把 Locality / Complementarity / Nonstationarity 写成可观测事实，而不是口号。
2. 全失败才救援 + 冻结 sibling 池，避开万能 harness 与耦合共演化的两头难。
3. 主表同时报 H0 与 Hrank1，强迫读者分清「政策变强了」和「推理还挂着外挂」。

问题与局限：主结果绑在 Qwen3.5-4B/9B 与 DeepSeek Harness 族编辑上；Router 依赖多 LLM 投票与验证轨迹，本身有成本；与前沿 GPT-6-Sol 等在 AppWorld 仍有差距（H0 上 4B 67.56 vs 82.82）。对工程的直接启发是：**别默认「最好的脚手架」要并成一个；先保住互补专长池，训练时只在底座没信号时借用，验收时必须看摘掉兄弟之后还剩多少。**

一句话：Harness-S 把兄弟 harness 当成可回收的探照灯，灯灭之后，路要还在原来的地板上。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
