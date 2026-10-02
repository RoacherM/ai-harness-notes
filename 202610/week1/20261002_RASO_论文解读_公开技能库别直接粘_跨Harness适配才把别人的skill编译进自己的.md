---
month: 202610
week: 1
date: 2026-10-02
type: 论文解读
slug: RASO
---

# 公开技能库别直接粘：跨 Harness 适配，才把别人的 skill 编译进自己的环境

你有没有过这种体验：GitHub / skills marketplace 上抄来一段「看起来完美」的 SKILL.md，挂进 Claude Code / Codex / 自家 Pi harness——工具名对不上、路径约定冲突、甚至把源 harness 的命令当成真理硬跑，分数不动或倒退。Korea University / KAIST / Meta AI 这篇 **RASO**（Retrieval-Augmented Skill Optimization via Cross-Harness Adaptation，arXiv:2609.38024）把锅钉死了：公开技能库里**绝大多数检索结果来自别的 harness**（SpreadsheetBench 上 92.9% 是跨 harness），直接粘贴等于把错误词汇表塞进目标环境。看完我的感受：这正是 Byron 关心的 **skills eng · multi-harness/ACP · marketplace 规模复用**——不是「再搜一条 skill」，而是 **RASI 无 rollout 初始化 + RASU 按失败检索更新 + Cross-Harness Adaptation 重写到目标对象/命令/单位**。

## 核心摘要

Agent skill 被定义为在给定 harness 下可复用的自然语言程序；百万级公开技能（如 GitSkills）已积累跨任务、跨模型、跨 harness 的过程知识，但现有优化器几乎只用**本任务 rollout** 迭代改 skill，忽略外部语料先验。RASO 把外部语料贯穿全程：共享管线做 **section 级 BM25 检索**，再用 **Cross-Harness Adaptation** 把源域专有名词与不可用过程剥掉，按目标任务 \(T\) 与 harness \(H\) 重写成可执行 lesson。两阶段互补：**RASI** 仅凭 \(T,H\) 与检索知识合成初始 skill \(s_0\)（零 rollout）；**RASU** 在失败轨迹上定位缺失知识，再检索–适配–写入 `skill.md`。四榜（OfficeQA / SpreadsheetBench / ALFWorld / WebShop）× 两模型（GPT-5.6-Luna、Qwen-3.5-9B）：RASI 已稳定压过「无检索初始化」与「直接拿检索 skill」；完整 RASO 相对 TextGrad / GEPA / SkillOpt / WikiSkill 全面领先——Luna 上 Spreadsheet 63.33 vs 最强基线 57.02（+6.31），Qwen 上 WebShop 24.73 vs 13.43（+11.30）。消融显示：Adaptation 开关在 RASI/RASU 两端都有大增益；检索段数 \(K=5\) 最优，语料越大越好但噪声会在 \(K=10\) 反噬。

## 论文信息

- **标题**：Retrieval-Augmented Skill Optimization via Cross-Harness Adaptation
- **作者 / 机构**：Jaewon Chu（Korea University）、Ji Soo Lee / Jihwan Park 等（KAIST）、Yunyang Xiong（Meta AI）；通讯 Hyunwoo J. Kim
- **语料**：GitSkills（Destefanis et al., 2026）等大规模外部 skill corpus
- **链接**：https://arxiv.org/abs/2609.38024 · https://arxiv.org/pdf/2609.38024
- **观察时间**：2026-10-02（HF Daily Papers 头条；skills · cross-harness · retrieval-augmented optimization）

---

## 🎯 为什么这件事值得写

技能优化两条旧路：（1）从自己的轨迹里 TextGrad/GEPA/SkillOpt；（2）检索后**原样复用**。前者探索不出 rollout 没见过的程序；后者在 harness 错配时把「别人的工具合同」灌进自己的 sandbox。RASO 的第三条轴是 **检索 + 编译**：先承认语料几乎全是跨 harness（图 1：Spreadsheet 上 92.9%），再强制适配层用目标 harness 的对象、命令、单位重写。对 Byron：这直接咬合 **SkillSeek / marketplace IR** 与 **MMP Manifest**——银行里的技能不是「粘贴即用」，而是 **IR → 适配 → 落盘**；也对照今早 [MetaSkill](./20261002_MetaSkill_论文解读_别把技能塞给Target_教Builder学元技能才配得动harness.md)：那边学「给谁编译」，这边学「从哪借过程知识再编译」。

## 🏗️ 机制：RASI / RASU 共享检索–适配，决策变量只有 skill 文本

设定：模型 \(M\) 与 harness \(h\) 全程冻结，唯一可优化对象是自然语言 skill \(s\)；优化覆盖**初始化**与**更新**两端，不假设外部已给好 \(s_0\)。

**Section 级检索。** 整篇 skill 噪声大、目标相似≠过程相关；按 heading 切段后，query agent 出 \(q\)，\(\mathcal{S}_q=\mathrm{BM25}(q,C,K)\)。

**Cross-Harness Adaptation（式 2）。** 适配 agent 对每个需求 \(c_i\) 产出 lesson \(\ell_i=\mathcal{A}_{\mathrm{adaptation}}(c_i,T,H,\mathcal{S}_{q_i})\)，三原则：（1）剥掉源域专名与目标 harness 无对应的步骤；（2）只服务 \(c_i\)；（3）工具/参数的具体断言须被 \(H\) 佐证——**正确性优先于虚假具体**。

**RASI（零 rollout）。** \(\mathcal{A}_{\mathrm{query\_init}}(T,H)\) 生成需求–查询对 → 检索–适配得 \(L_{\mathrm{init}}\) → \(\mathcal{A}_{\mathrm{skill\_init}}\) 合成 \(s_0\)，把每条 lesson 嵌进对应执行步。

**RASU（有 rollout）。** 执行 \(s_t\) → 从失败抽 textual gradient / 缺失知识 → 再检索–适配 → 写回 skill；下一轮用已提交 skill 重新 rollout。与「只靠本轨迹反思」的差别：探索空间被外部语料打开，但仍受目标 harness 词汇表约束。

## 🧪 关键证据

**主表（Table 1）。** 初始化段：Luna 上 RASI 相对 RFSI 的 OfficeQA 40.11→45.74、Spreadsheet 44.40→49.17；直接 SkillRouter 常接近甚至低于 No skill——**检索≠会用**。更新段：RASO 在两 backbone 上全面压过 TextGrad/GEPA/SkillOpt/WikiSkill；Qwen 上 ALFWorld +7.47、WebShop +11.30 尤其刺眼——弱模型更吃「外部过程 + 适配」。

**组件消融（Table 2，Luna）。** RFSI+RFSU：OfficeQA 40.70 / Spreadsheet 51.67；只换 RASU → 47.56 / 61.07；只换 RASI → 45.93 / 58.45；双开 → **49.03 / 63.33**（相对基线 +8.33 / +11.66）。初始化与更新的检索增益**互补**。

**Adaptation 开关（Table 3）。** RASI 无适配 41.86/41.43 → 有适配 45.74/49.17；RASU 同理 +2.33 / +5.47。固定 RASI 后换更新器（Table 4）：RASU 仍压过 WikiSkill/SkillOpt。

**\(K\) 与语料规模。** Figure 3：\(K=1\to5\) 上升，\(K=10\) 略掉——段太多噪声反噬。Figure 4：语料从 1%→100% 持续受益，虚线「无语料」显著更低。

**泼冷水。** 主数字绑 GitSkills + 四榜；适配本身是 LLM agent，成本进优化账本；跨 harness 成功依赖 \(H\) 描述是否诚实完整——Manifest 写糊了，适配原则（3）会「正确地什么都不敢写」。

## 🔬 最有意思的部分

1. **92.9% harness mismatch 是默认，不是边角。** 技能市场检索的先验分布就是「别人的 harness」——工程上必须假设错配。
2. **RASI 零 rollout 就能抬分。** 测试时 / 冷启动不必先烧轨迹；对齐「先有可执行初稿再迭代」。
3. **Adaptation = 编译器，不是摘要。** 剥名词、对齐命令单位、用 \(H\) 校验断言——这是 skills eng 的 IR→目标方言 步骤。
4. **SkillRouter 直接用检索 skill 经常无效。** 对照 [Skill-Use](./20261002_Skill-Use_论文解读_技能挂上了不等于会用_Trigger合规边界才测真使用.md)：挂上、检索到、会用、适配后可用，是四层不同能力。
5. **\(K\) 有甜点。** marketplace 越大越要细粒度检索 + 截断，而不是 top-50 整篇灌进上下文。

## 🤔 我的判断

定位：**跨 harness 的检索增强技能优化框架**（系统方法，把公开技能库从「复制粘贴」改成「检索–适配–落盘」）。亮点三：

1. 把 harness mismatch 量化成主矛盾，而不是脚注；
2. RASI/RASU 共享适配层，初始化与更新同一套编译纪律；
3. 相对强优化基线的稳定增益 + 清晰消融（Adaptation / \(K\) / 语料比例）。

局限：依赖语料覆盖与 \(H\) 规格质量；适配 agent 本身可错；未深入多 harness 同时服务（ACP Host）的路由问题。对 Byron：**(1)** skills marketplace / SkillSeek 落地时，**默认走 Cross-Harness Adaptation**，禁止「BM25 top-1 原文写入 Manifest」；**(2)** MMP / Pi：Rules 里写清工具合同与单位，适配层才有 \(H\) 可校验；**(3)** 对齐 MetaSkill——外部语料给的是过程知识，**编译权仍在 Builder/适配器**；**(4)** 与今早 Skill-Use 连读：Trigger/Compliance/Boundary 测的是「会不会用」，RASO 解决的是「借来的程序能不能变成可执行方言」。

一句话收束：**公开技能库默认来自别的 harness——RASO 证明：section 检索 + Cross-Harness Adaptation，才把别人的过程知识编译进自己的 skill，而不是把错词汇表粘进 sandbox。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
