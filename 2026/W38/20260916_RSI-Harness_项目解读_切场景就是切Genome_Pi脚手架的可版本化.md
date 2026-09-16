---
week: 2026-W38
date: 2026-09-16
type: 项目解读
slug: RSI-Harness
---

# 切场景就是切 Genome：RSI-Harness 把 Pi 的脚手架收成可版本、可分享的一包

你有没有过这种体验：同一个 coding agent，换一套系统提示、skills、MCP、权限策略，跑出来的效果判若两人。行业里管这套包裹模型的东西叫 harness（脚手架）。大家都承认它重要，也都在调——但这些配置往往散落在 `settings.json`、CLI 参数和「我记得上次那个 prompt 挺管用」里：**没法 diff、没法交接、换机器就漂，更谈不上复现。**

CosmosMind 的 RSI-Harness（产品名 RSIH，仓库 CosmosMind-ai/RSI-Harness）冲着这件事去：在 Pi coding agent 之上加一层叫 **Genome** 的配置对象，口号直接写进文档——**Switching contexts is switching Genomes（切场景 = 切 Genome）**。看完我的感受是：它几乎不发明新的 agent loop，真正值钱的是「把 harness 面收成可拷贝目录」的协议，以及从真实 session 历史长出 Genome 的那条自指流水线；如果你已经在走确定性 Pi harness（比如自研的 MMP），这是同赛道的产品化样本，不是又一个要你整仓迁移的框架。

## 核心摘要

RSIH **不 fork Pi Core**：agent loop、工具、模型、TUI、session fork/resume 全交给 `@earendil-works/pi-coding-agent`（发行钉在 `0.84.3`）。本仓库只做 Genome 适配层——把 instructions、tools、skills、commands、model、runtime、policies、integrations（extensions + MCP）、appearance、settings、keybindings、resources 共 **12 个组件**收成一个可分发目录。无 Genome 时 `rsih ≈ pi`（配置目录改成 `~/.rsih`）；有 Genome 时用 **inherit-by-default** 的 patch 叠配置：字段缺省则继承，显式 `null` 交还 Pi 默认，有值则覆盖（对象递归合、数组整换）。资源层可开 `resources.isolate: true` 关掉全局 skills/themes 自动发现——文档实测无关 skill 从几十个压到只留声明项，prompt 从约 36KB 掉到约 7KB。另有一个自指 Genome `harness-rsi`（命令 `gee`）：扫描 RSIH / Pi / Claude Code 等 session，用场景当尺子做证据筛选，**整份方案先写给人看，确认后才落盘**，再 `rsih genome validate`。所谓 RSI，改的是 harness 配置，不是模型权重。这不是「我们的 Agent 更强」的 demo 仓，而是一套把脚手架变成可版本对象的工程协议。

## 项目信息

- **名称**：RSI-Harness（RSIH）
- **维护者**：CosmosMind-ai
- **仓库**：https://github.com/CosmosMind-ai/RSI-Harness
- **栈**：TypeScript，Node ≥ 22.19，依赖 Pi coding agent + MCP SDK
- **观察时间**：2026 年 9 月（仓约 9/3 起公开，本文基于当时 main）

---

## 🎯 为什么这件事值得单独写

先交代分野。这两年 agent 产品卷的大多是「更能干活」：更强模型、更多工具、更花的 multi-agent 编排。另一条不那么吵但同样疼的线是：**同一模型换包一层 harness，体验完全变样，而这层东西却最难版本管理。**

社区常见三种态度：

1. **环境主义**：skills / MCP / hooks 越多越好，全局装满，靠模型自己挑；
2. **提示词主义**：一切写进巨大 system prompt，靠人肉迭代；
3. **确定性 harness**：显式声明本场景允许什么，禁止 ambient 泄漏（MMP 走的是这条）。

RSIH 明确站在第三种，并且多走了两步：一是把「本场景完整 harness 面」收成 **Genome 目录**（别人丢进 `~/.rsih/genomes/` 就能跑）；二是用 `harness-rsi` 从你的真实操作史里**提案**下一个 Genome，而不是空手上桌问「你想要什么 system prompt」。

上下文税是它反复强调的动机。skills 一多，名字和描述先占满 prompt，真正相关的流程反而被挤掉。`isolate: true` 不是小优化，是世界观：默认不继承全球技能海。

## 🏗️ 机制：Genome 是 patch，不是整仓重写

形式上看，Genome 是一组组件文件 + `genome.json` 清单，大致长这样：

```text
<name>/
  genome.json
  components/<id>.json
  contracts/<id>.dev.md
  skills/<skill>/SKILL.md
  extension/<name>.ts
```

合并语义是整套设计的骨架，值得单独记住：

| 字段状态 | 含义 |
|------|------|
| 缺省 | 继承 base，最终继承 Pi 默认 |
| `null` | 显式交还 Pi 默认 |
| 有值 | 覆盖；对象递归合，数组整换 |

含义是：你从不需要为了改 model 或加一个 MCP，就重写整份 harness。这和「复制一份巨型 settings 再手改」是两种工程文化。

两条不变量写进测试文化里：

1. 不带 `--genome` 时行为等于 `pi`（仅 configDir → `.rsih`）——降低迁移心理成本；
2. Pi 表面能配的开关，Genome 都要能配——有测试从 Pi 的 `.d.ts` 抽字段，漏接线就红。

种子 Genome 首次使用拷到用户目录；用内容哈希区分 current / stale / modified，避免静默覆盖本地改动。bundle 还可以 `base: 相对路径` 叠层（文档里承认相对路径解析仍有坑）。

## 🧪 GEE / harness-rsi：从行为长出配置

我觉得最有意思的不是 12 组件清单，而是自指那条线。

流程刻意不像「填问卷生成 prompt」：

1. **先问场景**（大白话，不急着对话框选项）
2. 扫描本机 session 目录（RSIH / Pi / Claude Code 等），按工作区聚合
3. 用 bash 先做直方图（工具、命令、热文件），再选择性读原文——控制上下文
4. 场景当尺子：证据强但与场景无关 → 丢掉；服务场景但证据薄 → 提问，不瞎编
5. **整份方案写给人看，确认后才写盘**
6. `rsih genome validate`

配套的 `pattern-to-component.md` 判据很硬：默认答案往往是 skill；上 MCP 门槛最高（要跨会话、要真有外部系统）；别把单次 bug corpus mirror 成永久记忆；别为了对称填满 12 个组件。这套「证据 → 组件类型」的表，比任何「自动生成 agent」营销页都更像能落地的方法论。

对照一下自我改进文献里的角色分野会更清楚：ShinkaEvolve / GEPA 一类是把 LLM 塞进大搜索当零件；VeRO / MetaHarness / HarnessOpt-Bench 关心的是 coding agent **端到端改 harness 代码**。RSIH 的 GEE 更偏「人在环的配置综合」——改的是 Genome 文件，且默认要人点头，不是无人值守刷 held-out。

## 🔬 最有意思的部分：它故意做得很「薄」

翻 `src/` 会有点「就这？」：cli、genome-loader、pi-projection、settings-layer，薄薄一层。价值几乎全在协议、文档和内置 Genome 样本（比如 `paperlab`：只声明少量组件的最小例子）。

这其实是优点。Fork Core 的 harness 最终都要跟上游赛跑；RSIH 选「投影到 Pi 公开配置面」，用测试锁住表面完整度。对读者的含义是：**你要偷的是 Genome 语义和 GEE 方法论，不是又一份 agent runtime。**

另一个反潮流点是 isolate。社区插件市场在教人「装更多」，RSIH 在教人「场景里只留该留的」。和 openai 把 skills 收成 plugins（安装/分发单元，内部仍按 skills / MCP / hooks 各走各的加载路径）可以对照着看：plugins 解决的是**分发与发现**；Genome 解决的是**场景切换与版本化配置面**。两者叠得上，但不是同一层问题。

## 🤔 我的判断

这篇（这个项目）的真实定位：**工程协议样本，不是方法突破论文，也不是跨运行时的通用 agent 平台。**

亮点我列三个：一是 inherit-by-default 的 Genome patch，把「改一个旋钮」从复制整仓降成写一个组件文件；二是 `isolate` 把上下文税当成一等公民，而不是事后靠模型硬扛；三是 GEE 的证据门槛表——从 session 直方图到「确认后才落盘」，比空谈 RSI 更可执行。

问题也有。仓很新（2026-09），社区 Genome 少，生产硬度未知；**绑死 Pi**，Claude Code / Codex Core 跑不了同一 Genome；相对路径 base、生态冷启动都是实打实的摩擦。如果你的主战场不在 Pi，把它当设计参考即可，犯不着为了 Genome 迁栈。

对已经在做确定性 Pi harness 的人（MMP 一类），启发是立竿见影的：Manifest 里的 skills / rules / MCP / hooks /「禁 ambient」几乎可以逐项映射到 Genome 组件；本地 `~/.mmp/pi` 之类的 session 目录，理论上就是 GEE 的饲料。下一步不是重写世界观，而是翻译字段、对齐 Pi 小版本差（例如 0.83 vs 0.84.3），先做一个最小 Genome 把「切场景」跑通。

最后一句话收束它的野心，其实文档已经写在标题里了：下一个问题不只是更好的 Agent，而是**能不能把「这一场景下的 Agent」收成别人可检出、可 diff、可回滚的一包配置**。RSI-Harness 给的就是这个包的一种形状。

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
