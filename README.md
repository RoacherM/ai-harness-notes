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
| 202609 | [week3](./202609/week3/) | [RSI-Harness](./202609/week3/20260916_RSI-Harness_项目解读_切场景就是切Genome_Pi脚手架的可版本化.md) · [ComposeCL](./202609/week3/20260916_ComposeCL_论文解读_持续学习机制组合起来才扛得住百任务记忆.md) · [LLM-as-Judge](./202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) · [Agora](./202609/week3/20260918_Agora_论文解读_加agent不等于加发现_Git_DAG当集体科研共享记忆.md) · [MoModels](./202609/week3/20260918_MoModels_论文解读_模型池越大越强_异质MAS里多经常变成更差.md) · [Meta-Harness](./202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md) · [Harness-Design](./202609/week3/20260918_Harness-Design_论文解读_脚手架不是一整块_组件要按模型和预算配.md) · [SoL-Pi](./202609/week3/20260918_SoL-Pi_论文解读_先省token再扩RSI_四个机制把Pi流量砍半.md) · [EvoSkill-GUI](./202609/week3/20260918_EvoSkill-GUI_论文解读_技能不是静态文档_部署时无训练自进化.md) · [EvoHarnessBench](./202609/week3/20260921_EvoHarnessBench_论文解读_脚手架越加越忘_非平稳性放进harness本身.md) · [Subagents-vs-Skills](./202609/week3/20260921_Subagents-vs-Skills_论文解读_技能该塞进主上下文还是开子代理.md) · [Code2Skill](./202609/week3/20260921_Code2Skill_论文解读_还没跑轨迹也能攒技能库_从两万仓库榨出百万条.md) · [GraphSkillEvo](./202609/week3/20260921_GraphSkillEvo_论文解读_技能别再写成一长串checklist_做成步骤图再进化.md) · [TrustReviewer](./202609/week3/20260921_TrustReviewer_论文解读_AI审AI审出来的裁判会塌缩_评分挤扁语义同质.md) |

## 单篇结构（见 TEMPLATE.md）

1. 钩子标题 + 「你有没有过这种体验」开场 + 看完感受  
2. `## 核心摘要`  
3. `## 论文信息` / `## 项目信息`  
4. `---` 后用 `🎯🏗️🧪📈🔬🤔` 等章节展开  
5. 以 `## 🤔 我的判断` 收束；文末固定互动 footer  

## License

笔记以 CC BY 4.0 共享；引用论文与项目请遵循其原许可证。
