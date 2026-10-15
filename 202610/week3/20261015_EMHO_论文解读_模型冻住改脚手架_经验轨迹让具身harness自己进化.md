---
month: 202610
week: 3
date: 2026-10-15
type: 论文解读
slug: EMHO
---

# 模型冻住改脚手架：经验轨迹，让具身 harness 自己进化

你有没有过这种体验：具身 agent 卡在门口来回转、抓同一坐标反复失败，你去翻轨迹却只看到稀疏的「动作无效」和最终成败——coding agent 至少还有测试红字、异常栈，这里连「哪一步害的」都不好指。于是大家习惯去训更大的 VLM，或者只改 skill / 恢复提示。**如果模型冻住，只允许改外围的规划、记忆、视觉工具编排与恢复逻辑，稀疏反馈下 harness 能不能自己进化？**

Korea University、Arkansas、Wisconsin–Madison 这篇 **EMHO**（arXiv:2610.08432）的回答是可以：内环跑任务攒经验轨迹，外环对照历史 harness 改动与结果做定向修订；多子任务共享一份脚手架时再用 **EMHO-Merge** 看 episode 级得失，修「何时启用」而不是盲目叠规则。看完我的感受：这是把 [Meta-Harness](../../202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) 那条「搜脚手架」思路搬进反馈更稀的具身世界，而且定性分析显示进化不只会写恢复口令，还会改进度监控、工具编排与视觉 grounding。

## 核心摘要

EMHO（EMbodied Agent Harness Optimization）固定底层视觉–语言模型 $M$，在 harness 空间 $\mathcal{H}$ 上最大化期望任务成功 $H^\star=\arg\max_H\mathbb{E}_{e\sim\mathcal{D}}[y_e(M,H)]$。内环在搜索集上执行当前 $H^{(i)}$，记录轨迹 $\mathcal{T}_e=(x_e,\{(o_t,p_t,z_t,a_t,f_t)\},y_e)$；外环用累计经验 $\mathcal{E}^{(0:i)}$ 与 harness 历史 $\mathbf{H}_{\mathrm{hist}}$ 提出候选，仅当搜索集成功率更高才替换父脚手架。EMHO-Merge 在联合优化时对比父–子：失败变成功为 gains $\mathcal{G}$、成功变失败为 losses $\mathcal{L}$，二者皆非空且该修订尚未修过时，用成对轨迹找可观察条件，限制行为触发时机（每条原修订最多一次 repair）。评测在 EmbodiedBench 导航与操作上，Qwen3.5-9B 与 Qwen3.8-27B-FP8，视觉工具含 Depth Anything 3、SAM 3 与 zoom，每任务 10 轮优化、搜索/留出对半。摘要称导航最高约 **+23.3%**、操作约 **+33.3%**（百分点量级见 Table 2：操作 Base 上 EMHO +33.3）；27B 操作搜索集平均 23.9→**44.2**，留出 26.4→**39.4**；9B 导航搜索 50.0→**63.0**、留出 48.7→**56.7**。全 harness 优化显著大于只改 skill；Merge 在联合搜索集平均上高于简单 Combined。价值定位：证明具身场景下「冻模型、进化脚手架」可行，并把优化对象从 skill/恢复扩到监控、工具与 grounding。

## 论文信息

- **标题**：EMHO: EMbodied Agent Harness Optimization via Experience Traces
- **作者**：Hyun Jung Lee, Jungtaek Kim, Jongwon Jeong, Tae-Eui Kam, Donghyun Kim（通讯）, Yong Jae Lee
- **机构**：Korea University · University of Arkansas · University of Wisconsin–Madison
- **项目页**：https://hyunjung00.github.io/emho/（正文写 “code can be found here”；抽取文本未给出独立 GitHub 仓库地址）
- **链接**：https://arxiv.org/abs/2610.08432 （2026-10-06；HF Daily Papers 有收录）

---

## 🎯 为什么这件事值得写

具身失败难诊断：长轨迹、弱反馈、延迟成败。现有自改进多锁在 skill、任务代码或运行时 critic（EmbodiSkill / VASO / ASPIRE / Zetta）。编码侧已有 Meta-Harness、Self-Harness、AutoSaddler 用执行痕迹改脚手架，但失败信号通常更显式。EMHO 问的是：**反馈更稀时，能否把更广的 harness 当成优化对象？**

这和 [ReHarness](../week2/20261012_ReHarness_论文解读_反思写进记忆就不再改_反向修订让harness状态也能吃到下游反馈.md)「脚手架状态也要吃下游反馈」、[EmbodiedCLBench](../week2/20261012_EmbodiedCLBench_论文解读_做过一道基础题就涨三倍_再做五道却不再涨_具身持续学习卡在经验提取.md)「经验提取卡住持续学习」、[AgentGarten](../week2/20261009_AgentGarten_论文解读_四轮写playbook顶过百万局RL_代码世界加神经渲染器让agent边玩边进化.md)「边玩边写 playbook」是同一条谱系：**别只烤权重，让外围规程从轨迹里长出来。** 也和 [EvoSkill-GUI](../../202609/week3/20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) 形成对照——EMHO 明确说只进化 skill 不够，还要改历史表示、重复检测与工具编排，让规则能被触发和强制执行。

| 路线 | 改什么 | 典型信号 |
| --- | --- | --- |
| 训更大 VLM / VLA | 权重 | 示范、RL |
| Skill / 恢复自进化 | 文本技能、critic | 轨迹反思、反例 |
| Coding Meta-Harness | 可执行脚手架 | 测试、异常 |
| **EMHO** | 规划/记忆/工具/恢复整栈 | 稀疏动作反馈 + episode 成败 |

## 🏗️ 机制：双环 + 得失感知的 Merge

**Harness 内容。** 以操作域为例，附录拆成四块：Planning（SKILL.md、system、planner、agent 循环）、Vision/tool（orchestrator、vision_policy、tool_select）、Memory/context（module_selector、memory、safe_context）、Validation（parser、repair）。导航同理。优化器只改允许文件，禁止硬编码 episode、偷看 simulator 隐状态或留出数据。

**EMHO 外环。** 候选 $\widetilde{H}^{(i+1)}=\mathcal{O}(H_{\mathrm{par}},\mathcal{E}^{(0:i)},\mathbf{H}_{\mathrm{hist}})$；仅当 $s(\widetilde{H})>s(H_{\mathrm{par}})$ 才晋升。接受与拒绝的轨迹都进历史，避免「只学赢家」。

**EMHO-Merge。**
$$
r^{(i)}=\mathbb{I}\big[|\mathcal{G}^{(i)}|>0\land|\mathcal{L}^{(i)}|>0\land\text{repair unused}\big]
$$
$r=1$ 时下一候选优先修「何时/如何」应用上一路修订，而不是另起炉灶；修复仍挂在当前父脚手架上，使被拒候选的经验也能指导下一步。

**实验协议。** EmbodiedBench 导航（Base/Common/Complex/Visual/Long）与操作（Base/Common/Complex/Visual/Spatial）；每任务搜索/留出对半；10 次迭代；主表外实验默认 27B。

## 🧪 关键证据

**主结果摘录（Table 1，平均成功率 %）。**

| 设定 | 导航 Search | 导航 Held-out | 操作 Search | 操作 Held-out |
| --- | --- | --- | --- | --- |
| Initial · Qwen3.5-9B | 50.0 | 48.7 | 9.7 | 10.7 |
| EMHO · 9B | **63.0** | **56.7** | **18.3** | **21.0** |
| Initial · Qwen3.8-27B | 54.7 | 66.0 | 23.9 | 26.4 |
| EMHO · 27B | **67.3** | **75.3** | **44.2** | **39.4** |

27B 操作 Base 搜索从 16.7 到 50.0；导航上进化后的 9B 搜索平均可与部分更大公开模型同量级。相对初始 harness，Table 2 给出搜索集百分点增益：导航 EMHO 在 Base/Common +16.7，高于 skill-only 的 +10.0；操作 Base 上 EMHO **+33.3** vs skill-only +16.7，Common +29.2 vs +8.3。

**One-shot 前沿改脚手架。** Claude Opus-5 / GPT-5.6 Sol 各看初始 harness + 代表性例子写一版；能抬初始，但两域都不如 EMHO。作者观察：one-shot 爱给「卡住就换策略」这类宽指导，EMHO 会落成可执行条件——例如阻挡后终止当前计划剩余动作、禁止同向重试、基于更新观察重规划。

**EMHO-Merge（Table 3，搜索集 Avg）。** 导航：Initial 54.4 · Separate 65.6 · Combined 61.1 · **Merge 67.8**。操作：Initial 25.9 · Separate 47.2 · Combined 34.7 · **Merge 39.8**（仍低于 Separate，但高于乱合并）。例子：Base 导航学到的「阻挡后先横移」在长程轨迹里会反复触发撞近障；Merge 把避障限制到有阻挡证据时。

**泼冷水（作者 Limitations）。** 搜索集上最好的迭代不一定在留出最好——后期可能过拟合所见轨迹；选 harness 迭代时要显式考虑泛化。Merge 在操作留出平均（33.8）仍低于 Separate（40.7），共享一份脚手架有代价。

## 🔬 最有意思的部分

1. **恢复编辑先来，非恢复后到。** 早期提案几乎都是失败响应；进度监控、工具选择、grounding 类编辑要等 harness 先有显式失败处理，才更占份额——稀疏信号下，最可观察的失败是最近的优化靶。
2. **规则越改越细，不是堆忠告。** 导航恢复链：不重复被挡动作 → 多方向被挡后旋转 → 按阻挡方向、目标朝向、绕障距离选响应。优化器在把泛反应炼成情境策略。
3. **有效但无效的动作也要侦测。** 无 invalid 信号时：导航连续 ≥5 次旋转变少；操作相同重规划约 21%→18%，同坐标重复抓取约 30%→26%。
4. **工具编排跟着不确定性类型走。** 挡后 depth 使用约 70%→近 100%，随后有效动作约 44%→68%；操作里对堆叠/容器等高度敏感交互会主动调 depth。
5. **Grounding 变细。** 操作场景描述列出 ≥3 物体约 8%→22%；空间关系任务初始计划错选物体 50%→34%。导航在推理中显式点名目标后到达率升、越过目标走远的情况降。

## 🤔 我的判断

定位：**方法突破 / 具身版 harness RSI**。它不是又一篇「加恢复 prompt」，而是把可编辑面扩到监控、记忆、工具与验证，并在多子任务共享时给了得失感知的 Merge。

亮点三条：

1. 冻模型只进化脚手架，在 EmbodiedBench 导航与操作上搜索与留出双升；
2. 全 harness ≫ skill-only，说明收益来自「让规则可被检测、触发、强制」；
3. EMHO-Merge 把跨子任务冲突从「平均掉分」变成可修的触发条件问题。

问题与局限：搜索–留出过拟合；Merge 未全面赶上分任务 Separate；10 轮、单基准、主要 9B/27B；优化器本身是强模型提示工程，成本与稳定性未充分披露；项目页有、独立仓库地址在抽取中不显。

对工程的直接启发：**(1)** 具身/GUI agent 优先考虑「冻模型、进化 harness」，尤其当反馈只有成败时；**(2)** 优化对象别停在 SKILL.md——把重复检测、失败反馈入上下文、工具策略一并纳入可编辑面；**(3)** 多场景共享脚手架时，记录 episode 级 gains/losses，用成对轨迹修触发条件，而不是继续盲合并；**(4)** 用搜索集选最终 harness 前，务必看留出曲线，防后期过拟合；**(5)** 与 [COAP](../week2/20261014_COAP_论文解读_具身世界当图灵机_状态写进代码决策也写代码_测试时不再调模型.md) 对照：一个把世界与决策沉成代码、一个把外围规程用轨迹进化——都在减测试时对大模型的依赖，但杠杆点不同。

一句话收束：**模型可以冻住，但 harness 不该冻住；EMHO 证明，在稀疏具身反馈里，只要把经验轨迹和修订史喂给外环，脚手架自己能从「卡住就重试」长成会监控、会调工具、会改 grounding 的操作规程。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
