---
month: 202610
week: 1
date: 2026-10-02
type: 论文解读
slug: FloWright
---

# 别只训工作流生成器：把 workflow 当 harness，角色才能共进化

你有没有过这种体验：花大力气 RL 训出一个「会画多 agent 图」的 Generator，上线后执行端照旧踩坑——因为**结果分数是整条 workflow 一个标量**，下游工人、Inventor、工具封装从没从失败里学到自己那一脚。William & Mary / NEC / UIUC 这篇 **FloWright**（It Takes Workflows to Evolve Better Workflows，arXiv:2610.01026）把 workflow **本身**定义成共享 harness：同一套执行–打分信号，既评测、又训任意角色、还在测试时做 meta distillation。看完我的感受：这是对 Byron **control-plane · multi-agent · skills 共进化** 的直接补丁——**共进化更多角色（+5.03%）明显强于只优化单一角色（+2.83%）**，且不靠额外 critic / 人工标注 / 重复执行来做归因。

## 核心摘要

多 agent 工作流把复杂任务拆成图 \(G=(V,E)\)，节点来自共享池 \(\Omega\)（agent / tool / skill，或粗粒度 capsule）。既往「学造更好 workflow」几乎只训 Generator，其余构建/执行角色冻结；扩展训练又撞上耦合与稀疏全局分。FloWright 的解法：（1）**workflow-as-harness**——\(h(G,x)\in[0,1]\) 同时是评测、训练信号与测试目标；（2）从 harness 已记录的执行轨迹 \(\tau\) **零额外成本**做角色级 credit localization：角色 \(\rho\) 只为自家节点失败比例付负信用 \(c_j^\rho\)；（3）**分层结构感知奖励** \(h_j^\rho=w_f f+w_v v^\rho+w_e e+w_a a+w_c c_j^\rho\)，从「输出可解析 → 角色贡献合法 → 跑出答案 → 答案正确」搭梯子；（4）四种演化模式——单角色自进化、agent–skill 共进化、上下游共进化、多 agent 共进化。配套 **DataWright** 把单 agent 已能做的 12 数据集 / 7 域硬化成 workflow 级任务（shared / paired / decoy 等策略，\(\ell\in\{3,5\}\)，44 评测臂）。小开源模型上训练时相对未训 FloWright 最高约 **+7.41%**；4B 上多角色共进化 overall **+5.03%** vs 单角色 **+2.83%**；测试时 meta distillation（5-shot）再叠 train-time，overall 最高约 **+3.21%**。项目页：https://xhguo7.github.io/FloWright/。

## 论文信息

- **标题**：It Takes Workflows to Evolve Better Workflows（FloWright）
- **作者 / 机构**：Xuehang Guo（William & Mary）、Haoyu Wang / Haifeng Chen（NEC）、Yangyi Chen / Zhenhailong Wang（UIUC）等；共同指导 Zhenhailong Wang、Qingyun Wang
- **项目页**：https://xhguo7.github.io/FloWright/
- **链接**：https://arxiv.org/abs/2610.01026 · https://arxiv.org/pdf/2610.01026
- **观察时间**：2026-10-02（HF Daily Papers；multi-agent workflow · harness · co-evolution）

---

## 🎯 为什么这件事值得写

「多 agent 涨分」近年常被复现打脸：在单 agent 已能处理的数据上，workflow 增益很小甚至为负。FloWright 同时打两处：（A）**优化对象错了**——只训画图的人，不训干活的人；（B）**数据难度错了**——要用 DataWright 把题硬化到「必须分工才划算」。归因又不走昂贵 LLM judge 链，而是 harness 轨迹里读「谁的节点根因失败」。对 Byron：这与 [Raven](./20261001_Raven_论文解读_别再手搓更强harness_Harness的Harness才把异构agent当可组合单元.md) 的 Host、[MetaSkill](./20261002_MetaSkill_论文解读_别把技能塞给Target_教Builder学元技能才配得动harness.md) 的 Builder 同一光谱——**控制面信号要能落到具体角色**，而不是一个全局 0/1。

## 🏗️ 机制：上下游 + 池动态 + 分层信用

**上游建、下游跑。** Generator \(\pi_g\) 产出 \(G\)；harness \(H\) 执行并打分。上游可挂 Inventor skill、多 agent 协同；池粒度可细（agent∪tool∪skill）或粗（capsule）；池可**动态扩容**——缺能力时当场发明组件（消融关动态池会掉分）。

**拓扑非线。** 顺序 / 并行 / 分支调和 / 迭代直至停——Generator 按任务选控制流，而不是永远 chain。

**Credit（式 2）。** 角色 \(\rho\) 对节点子集 \(V_j^\rho\) 负责；从 \(\tau_j\) 收回根因失败集 \(F_j^\rho\)，\(c_j^\rho=-|F_j^\rho|/|V_j^\rho|\in[-1,0]\)。无失败则 0——**不需要第二模型当裁判**。

**分层奖励（式 3）。** 同一梯子，各角色用不同权重实例化（附录表）：先逼输出可解析/合法，再逼整图跑通与正确，最后用局部信用切开耦合。

**训练 / 测试同一 harness。** 训练：任意 policy-gradient（PPO / GRPO / DAPO / CISPO 可换）最大化 \(\mathbb{E}[h^\rho]\)。测试：优化可复用 prior \(P\)（内部经验 / 强教师 / few-shot meta），**权重不动**仍抬分。

## 🧪 关键证据

**四模式主表（Table 1）。** 未训 FloWright 已相对「带工具技能的单 agent」大幅领先（9B overall 47.50 vs 30.75）——小模型靠建/跑 workflow 就能超过「同等大小单打」。训练后：单角色 (a) 4B +2.83 overall；agent–skill (b) +4.06；上下游 (c) +3.06；多 agent (d) **+5.03**，且 in-dist / OOD / OOD-domain 三区均为正。共进化角色越多，增益越大。

**测试时（Figure 3）。** 内部 distill +0.93、外部教师 +1.10；5-shot meta 最大约 +2.58（1-shot 反低于未训——先验证据不足会害人）；train-time + 5-shot 合到约 **+3.21** overall。

**泛化。** 优化任意单角色都涨；4B 角色的增益可迁移到其余角色跑 9B，甚至超过全 9B 基线；RL 算法、奖励项、池动态、拓扑选择均有消融支撑「范式可换骨」。

**泼冷水。** 主训在 paired@\(\ell=3\) 四数据集；judge 仍用 GPT-5.6-luna 做部分任务类型；硬化后的「workflow 必要」不等于生产里的真实组织成本；credit 依赖 harness 能标根因节点——轨迹日志不完整时公式会退化。

## 🔬 最有意思的部分

1. **Harness 三重身份。** 同一 \(h\)：榜、梯度、测试目标——少一套「训练用奖励模型、上线用另一套指标」的漂移。
2. **归因零额外执行。** 对照「再跑 critic / LLM judge 归因」路线，这里读的是 harness 本就该记的 trace。
3. **DataWright 把评测诚实化。** 先承认单题数据测不出 multi-agent，再硬化——比硬吹 MAS 叙事干净。
4. **共进化 = 互为课程。** 上游变强改变下游见到的分布，信用切开避免「全员背锅」。
5. **1-shot meta 害、5-shot 助。** 测试时记忆也要剂量——呼应「few-shot 不是越多越好，是证据够不够泛化」。

## 🤔 我的判断

定位：**把 workflow 升格为共享 harness 的多角色自/共进化范式 + DataWright 硬化评测**（系统方法，偏控制面与信用分配）。亮点三：

1. 明确批评「只训 Generator」并给出可扩展到任意角色的同一信号；
2. 结构感知信用不引入额外模型/标注/执行；
3. 用硬化数据与四模式表证明「共进化 > 单进化」，且 OOD 仍为正。

局限：硬化任务仍由规则拼装；根因标注质量依赖实现；生产组织/权限/计费未建模。对 Byron：**(1)** 多 harness / ACP 里，Host 的打分应能**切片到工人节点**，否则 RSI 只会卷提示词；**(2)** skills 与 agent 应允许 **agent–skill 共进化**（模式 b），别把 SKILL.md 当冻结文物；**(3)** 对齐 MetaSkill：Builder/Generator 与 Target/Downstream 要共享可验证 harness，而不是各玩各的奖励；**(4)** 评测先问「单 agent 是否已饱和」——饱和则先 DataWright，再谈 MAS 涨分。

一句话收束：**只训工作流生成器，等于让其他角色白干活却永不学习——FloWright 证明：把 workflow 当共享 harness，用分层结构信用让角色自进化或共进化，且共进化更多角色更赚。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
