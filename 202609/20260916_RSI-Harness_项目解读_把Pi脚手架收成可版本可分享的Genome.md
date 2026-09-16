# 把 Agent 脚手架收成可版本、可分享的一包：RSI-Harness（RSIH）在干什么

你有没有过这种感觉：同一个 coding agent，换一套 system prompt、skills、MCP、权限策略，表现判若两人——但这些配置散落在 `settings.json`、CLI 参数和「我记得上次那个 prompt 挺管用」里，没法 diff、没法交接、也没法复现。CosmosMind 的 **RSI-Harness（RSIH）** 冲着这件事：在 **Pi coding agent** 之上加一层叫 **Genome** 的配置对象——**切场景 = 切 Genome**。

读完我的判断是：它对「确定性 Pi harness」路线（比如你自己的 MMP）几乎同构，真正值钱的是 **Genome 协议 + 从 session 历史生成 Genome（GEE / harness-rsi）**，而不是又一个自研 agent loop。若你还不在 Pi 生态，当设计样本看即可，不必整仓迁移。

## 核心摘要

RSIH **不 fork Pi Core**：agent loop、工具、模型、TUI、session fork/resume 全交给 `@earendil-works/pi-coding-agent`。本仓库只做 Genome 适配层——把 instructions / tools / skills / MCP / policies / resources 等 **12 个组件**收成一个可拷贝目录。无 Genome 时 `rsih ≈ pi`（配置目录改成 `~/.rsih`）；有 Genome 时用 inherit-by-default 的 patch 叠配置。另有一个自指 Genome `harness-rsi`（命令 `gee`）：读你的真实 session，提案 skill/tool/MCP/memory，**人确认后才落盘**。所谓 RSI，改的是 harness 配置，不是模型权重。

## 信息卡

- **类型**：项目
- **名称**：CosmosMind-ai/RSI-Harness（产品名 RSIH）
- **链接**：https://github.com/CosmosMind-ai/RSI-Harness
- **观察时间**：2026-09（仓建于 9/3，本文基于当时 main）
- **栈**：TypeScript，Node ≥ 22.19，钉 Pi `0.84.3` + MCP SDK
- **主题**：harness 配置 / Genome / skills / MCP / session 取证

---

## 它在解决什么问题

今天调 agent，配置通常是：

- 全局 settings + 项目 flags + 粘贴 prompt + 脑子里的偏好
- 不可版本、不可分享、换机器就漂
- skills 一多，上下文被无关 skill 名占满（上下文税）

RSIH 的答案：把「这一场景下的完整 harness 面」收成 **Genome 目录**，别人丢进 `~/.rsih/genomes/` 就能跑。

## 机制要点

### 1. 不变量：不 fork Core

文档与测试强调两条：

1. 不带 `--genome` 时行为等于 `pi`（仅 configDir → `.rsih`）
2. Pi 能配的开关，Genome 都能配——`pi-surface.test.ts` 从 Pi 的 `.d.ts` 抽字段，漏接线就红

### 2. Genome = 12 组件的 patch

组件包括：instructions、tools、skills、commands、model、runtime、policies、integrations（extensions + MCP）、appearance、settings、keybindings、resources。

合并语义（最关键）：

| 字段状态 | 含义 |
| --- | --- |
| 缺省 | 继承 base，最终继承 Pi 默认 |
| `null` | 显式交还 Pi 默认 |
| 有值 | 覆盖；对象递归合，数组整换 |

所以「只改 model」不会清空 system prompt 或工具集——**你从不需要为改一个旋钮重写整个 harness**。

### 3. 可分发单元是自洽目录

```text
<name>/
  genome.json
  components/<id>.json
  contracts/<id>.dev.md
  skills/<skill>/SKILL.md
  extension/<name>.ts
```

种子（发行版内置）首次使用拷到 `~/.rsih/genomes/`；用内容哈希区分 current / stale / modified，避免静默覆盖用户改动。

### 4. `resources.isolate: true`

关掉 Pi 对全局 skills/themes 的自动发现。文档实测：无关 skill 从几十个压到只留 Genome 声明的，prompt 从约 36KB → 7KB。这和「skills 越多越好」的社区习惯对着干，方向正确。

### 5. GEE / harness-rsi：从行为长出 Genome

流程不是问「你要什么 system prompt」，而是：

1. 先问场景（大白话，不急着对话框选项）
2. 扫描 RSIH / Pi / Claude Code 等 session 目录，按工作区聚合
3. bash 先做直方图（工具、命令、热文件），再选择性读原文
4. 用场景当尺子：证据强但与场景无关 → 丢掉；服务场景但证据薄 → 提问
5. **整份方案写给人看，确认后才写盘**
6. `rsih genome validate`

`pattern-to-component.md` 里的判据表很硬：MCP 门槛最高、skill 是默认答案、别 mirror 单次 bug corpus、别为对称填满 12 个组件。

## 和我的栈怎么对照

| 我这边 | 可借鉴 / 可忽略 |
| --- | --- |
| **MMP（Make My Pi）** | 几乎同赛道：显式 Manifest、禁 ambient、钉 Pi 版本。MMP≈自研装配层；RSIH≈把装配收成可交接 Genome + 从 session 生成。可把 Manifest 字段映射到 Genome 组件；`~/.mmp/pi` session 可喂 GEE。注意 Pi 版本差（MMP 0.83 vs RSIH 0.84.3）。 |
| **Wayne-Skills** | Genome 的 skills 组件 + isolate = 场景级技能包，而不是全局技能海。 |
| **DeepSeek harness / ACP** | 弱相关——RSIH 绑 Pi，不是跨 Codex/Claude 的通用插件市场。学的是配置语义，不是 runtime 迁移。 |
| **TeamAI / openai plugins** | 同属「分发层」；RSIH 更强调单 agent 的场景切换与证据生成。 |

## 局限

- **绑死 Pi**：换 Claude Code / Codex Core 跑不了同一 Genome。
- 仓很新（2026-09），社区 Genome 少，生产硬度未知。
- `src/` 很薄：价值在协议与方法论，不在自研 loop。
- bundle 层叠 `base: 路径` 有相对路径解析已知坑。

## 决策

**值得深挖（设计层）/ 不必整仓迁过去。**

硬理由：它把「确定性 harness 配置」产品化成可版本对象，并用 session 取证自动长配置——这正是 MMP 积累要去的下一层；迁移成本是翻译 Manifest→Genome，不是重写世界观。

## 若继续跟，先看哪几处

1. `docs/genome/README.md` — 合并语义与分发
2. `docs/genome/harness-rsi.md` + `skills/genome-authoring/pattern-to-component.md` — 怎么从历史长出 harness
3. 内置 `paperlab` — 「只声明少量组件」的最小样本
4. 对照自己的 `mmp.json`：skills / rules / MCP / hooks / isolate 各落哪一格

---

*归档说明：基于公开 README / docs / 源码树与 2026-09-15 前后的调研笔记整理。*
