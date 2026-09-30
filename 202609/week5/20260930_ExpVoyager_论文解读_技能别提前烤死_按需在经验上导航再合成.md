---
month: 202609
week: 5
date: 2026-09-30
type: 论文解读
slug: ExpVoyager
---

# 技能别提前烤死：按需在经验上导航，再合成给执行器

你有没有过这种体验：agent 跑完一堆任务，把轨迹摘要成 `skill.md` 存进库——下次任务来了 top‑k 检索一塞。结果要么摘要时把后来才关键的条件裁掉了，要么相似轨迹里塞满实例细节、真正可迁移的步骤藏在「看起来不像」的失败局里。延世大学这篇 ExpVoyager（arXiv:2609.32630）把技能合成改写成 **在原始经验上的动态导航**：curator 按当前任务主动搜不同视图/分辨率，边看边维护 Navigation State，最后再写出给 **冻结 executor** 用的 skill。看完我的感受：这正打在 Byron 的 **skills / harness / 经验复用** 三角——作者开篇就把 agent skill 定位为 harness 运行时供给层，并显式对比「预构建技能丢失大部分未来有用知识」。

## 核心摘要

ExpVoyager = **Navigable Interface**（轨迹级 / 步骤级多视图 + `search_exp` / `inspect_traj`）+ **Navigation State** \(Z_r=(K_r,Q_r)\)（已沉淀的目标相关程序知识 + 仍待查的开放问题）+ **任务时 skill 合成**。预实验在 ALFWorld 上构造 80 个经执行验证的 oracle skill：现有预构建方法相对原始经验保留的 oracle 知识多数丢失；相似度检索加深 \(k\) 时知识 recall 上不去、precision 崩。主实验以 Qwen3.5-9B 为 curator：同骨干 executor 上 ALFWorld SR 63.7 vs 最强基线 SkillTTA 51.1 / AWM 45.6；WebShop score 61.2；ScienceWorld score 47.0；换 Gemma4-31B executor 仍领先（ALFWorld 69.6）。可与已有技能协同做 on-demand 精炼，并随经验库增大持续拉开差距。代码：https://github.com/tommyEzreal/ExpVoyager

## 论文信息

- **标题**：ExpVoyager: Direct Experience Navigation for Dynamic Agent Skill Synthesis
- **作者**：Kwangwook Seo、Dongha Lee†（† correspondence）
- **机构**：Yonsei University
- **链接**：https://arxiv.org/abs/2609.32630 · https://arxiv.org/pdf/2609.32630 · 代码 https://github.com/tommyEzreal/ExpVoyager
- **观察时间**：2026-09-30（HF Daily Papers；skills / harness 经验层）

---

## 🎯 为什么这件事值得写

从轨迹学技能（AWM、RBank、Trace2Skill、SkillTTA 等）已成自进化 agent 标配，但主流是 **先抽象、后检索**：在未知未来需求时就把经验压成固定程序知识。作者用 claim 级覆盖量化：预构建技能相对 raw experience 的 oracle 知识保留很差；被动 top‑k 也装不满目标所需知识。

重新提问：**如何按当前任务，从原始经验动态合成技能？** 答案不是「再多召回几条轨迹」，而是让 curator **自己决定去哪看、看多细**。对 Byron：这和 JITMem「读时策展」、SkillPivot「偏航点再改技能」同属 **延迟绑定**——别在写入时赌未来；也和 MMP/Pi 的 Manifest 技能供给同层：运行时到底塞哪份程序知识，可以是导航结果而不是静态文件。

## 🏗️ 机制：可导航接口 → 状态闭环 → skill.md

**问题形式。** 经验库 \(E=\{\tau_i\}\)，目标任务 \(x\)；curator \(\phi\) 产出 \(S_{E,x}\)，冻结 executor \(\theta\) 在 \((x,h_t,S_{E,x})\) 下行动。

**Navigable Interface。** 轨迹不只当检索原子：
- **轨迹级视图**：动作序列、结局、任务元数据——找大策略与成败模式；
- **步骤级视图**：观测 / 记录推理 / 动作 / 即时结果 + 邻域——找条件与局部因果。

导航动作：`search_exp`（指定视图层级、字段、正则、条数；命中返回完整记录以保上下文）与 `inspect_traj`（从记录引用展开整条源轨迹）。

**Navigation State。** 每轮观测后更新 \(K_r\)（区分：源场景特有事实 vs 可迁移程序关系 vs 执行时仍需核对的条件）与 \(Q_r\)（把知识缺口变成下一轮要回答的问题）。选一个最能改善最终可执行技能的开放问题，再选导航动作——**访问与解释互相引导**。

**合成。** 预算用尽或 FINISH 后，\(\phi_{\mathrm{synth}}(Z_R,x)\) 写成 skill.md：保留未决目标条件作执行时检查，失败课作阶段警告/恢复指引。

## 🧪 关键证据

**预实验。** 80 oracle skills + 300 源轨迹标注：预构建方法知识保留远低于 raw；检索范式知识 recall 低，加大 \(k\) 换来 precision 急跌。导航在相近平均访问源数下更高覆盖。

**主表（Qwen3.5-9B curator）。** Executor=同骨干时 ALFWorld SR：Base 41.4 → AWM 45.6 / Trace2Skill 45.4 / SkillTTA 51.1 → **ExpVoyager 63.7**（步数反而低于若干基线）。WebShop score 61.2、ScienceWorld 47.0，均超对照。Gemma4-31B executor：ALFWorld 69.6、WebShop 66.7、ScienceWorld 63.7。消融去掉 Navigable Interface 或 Navigation State 均掉点——说明不是「多轮随便搜」。

**扩展。** 在线自进化可不依赖预收集库；经验空间放大时优势继续拉开；可与已有技能叠加做按需精炼；访问预算敏感但主动导航比被动加深 \(k\) 更划算。

**泼冷水。** 主战场是 ALFWorld / WebShop / ScienceWorld，不是 SWE/MCP 重仓库；curator 调用本身有延迟与 token 税；失败案例显示「导航找对了顺序约束，合成时弄丢」——最后一跳 skill 写作仍脆。

## 🔬 最有意思的部分

1. **预构建 = 不可逆有损压缩。** 用 oracle 覆盖把「摘要技能」的信息损失测出来，而不是只比下游 SR。
2. **相似 ≠ 程序相关。** 关键线索可能在表面不像的轨迹；导航用问题驱动检索，而不是 embedding 邻居。
3. **源事实 vs 可迁移规则 vs 执行时检查。** Navigation State 强制三分法——咖啡桌上有报纸 ≠ 当前房也有咖啡桌；技能应留「执行时再验」。
4. **Curator / Executor 角色分离。** 小 curator 导航、冻住的执行器干活——和「主代理塞满经验」比，更像 harness 插件。
5. **和 TraceDance 互补。** TraceDance 把坏行为冻成评测；ExpVoyager 把好坏经验导航成运行时技能——一个外环测，一个内环供。

## 🤔 我的判断

定位：**任务时、导航式的 agent skill 合成框架**（多分辨率经验接口 + 状态引导 + 冻结执行器）。亮点三：

1. 把「技能该何时抽象」从写入时改到需求时，并有预实验支撑；
2. Navigable Interface / Navigation State 消融干净；
3. 跨 curator–executor 配置与经验规模都显示增益。

局限：尚未证明搬到 coding-agent / MCP 长轨迹同样便宜；合成步仍可能丢掉已找到的约束。对 Byron：别默认「轨迹 → 永久 skill 文件 → 检索」一条路走到黑——对高价值任务允许 **临时导航 skill**（用完可丢或再沉淀）；技能库索引至少暴露步骤级字段（工具名、错误串、决策理由），好让 curator/`search_exp` 类接口能用。和现有 Anthropic-style skills 共存时：静态 skill 当先验，ExpVoyager 做 on-demand 修订。评测别只报 SR：加「oracle 知识保留 / 导航预算 / 合成后是否丢顺序」切片。

一句话收束：**未来任务的需求在摘要时还不存在**——与其提前烤死技能，不如让 curator 在经验海上按需航海，再临时造船给执行器。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
