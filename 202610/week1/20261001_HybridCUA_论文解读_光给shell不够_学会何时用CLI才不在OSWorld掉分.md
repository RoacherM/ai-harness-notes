---
month: 202610
week: 1
date: 2026-10-01
type: 论文解读
slug: HybridCUA
---

# 光给 shell 不够：学会何时用 CLI，才不在 OSWorld 上白掉十几分

你有没有过这种体验：给 coding agent 开终端，文件读写、批量改格式一瞬间搞定；可把同一套「截图 + 鼠标键盘」的 computer-use agent 扔进 OSWorld，再塞一个 bash，分数反而塌——有的模型几乎不用 CLI，有的乱用命令早死。浙大 / 北大 / 清华这篇 **HybridCUA**（arXiv:2609.38008）钉死一点：**瓶颈不是有没有 shell，而是模型不知道何时、如何在 GUI 与 CLI 之间编排。** 看完我的感受：这正好接 Byron 的 **coding-agent / CUA toolchain · sandbox · skills** 线——比「再给每个 App 焊一套 API」更可扩展，也比「裸开终端」更需要训练信号。

## 核心摘要

HybridCUA 主张下一代 computer-use agent 应 **GUI 保通用 + CLI 保效率**，并用数据与奖励把「接口选择」变成可学能力。作者建可扩展流水线，产出三类轨迹：GUI only（开源 UI 轨迹改写成统一 pyautogui bash）、CLI only（应用文档蒸馏成 CLI skill，Claude Code harness + 仅 shell 采成功 rollout）、interleaved（自由切换 + GUI 段改写为等价 CLI 再回放）。得到 **HybridCUA-8K**：约 **5K** SFT 轨迹 + **3K** 带「CLI 是否明显更优」标签的可验证 RLVR 任务。训练两阶段：混合轨迹 SFT，再在双接口环境做 GRPO，轨迹级 \(R_{\mathrm{CLI}}\)（成功且 CLI 使用与任务标签一致）+ 步级 \(r^{\mathrm{exec}}\)（shell 级失败 −1）。**HybridCUA-9B**（Qwen3.5-9B）OSWorld **53.6%**（相对基座 **+14.8 pp**），步数 31.6→14.0；裸加 CLI 反把同基座打到 18.4%。WindowsAgentArena **36.0%**（+4.0），OSWorld-MCP **47.1%**；训练只见 Linux shell 仍会发 PowerShell——迁移的是「何时交给 shell」而非背命令。

## 论文信息

- **标题**：HybridCUA: Learning to Orchestrate GUI and CLI for Computer-Use Agents
- **作者**：Tongbo Chen*、Junbo Niu*、Zhengxi Lu、Niu Lian、Fei Tang、Yuchen Yan、Yike Hong、Yong Du、Yizhou Liu、Bofan Chen、Yongliang Shen†（* 共一；† 通讯）
- **机构**：浙江大学；北京大学；清华大学
- **链接**：https://arxiv.org/abs/2609.38008 · https://arxiv.org/pdf/2609.38008（文中标注将释出轨迹 / RLVR / 训练管线 / 模型）
- **观察时间**：2026-10-01（HF Daily Papers；CUA × CLI 编排）

---

## 🎯 为什么这件事值得写

GUI-only CUA 通用但长序列易级联出错；GUI+应用 API（ToolCUA / UltraCUA 等）快，却要按应用工程化，难扩展。Shell 随 OS 自带，能把一长串点击压成一条命令——coding agent 早已证明这一点。关键反直觉事实：**四类代表性 agent 一旦暴露 CLI，OSWorld 掉 2.5–11.5 pp**；有的 CLI 步占比只有 0.15%–15%。缺口有二：数据里 GUI / 终端 / 代码各自成仓，看不到「在截图上下文里下命令」；监督对接口选择失明——模仿只看局部像不像，结果奖励只看成不成。HybridCUA 把 **接口选择写进数据标签与 RL 奖励**，相对 RecreationWorld「只验结果、接口仍靠模仿」更显式。对 harness：这是 sandbox 里 **统一动作语法 + 可验证任务 + 接口级信用** 的完整样本。

## 🏗️ 机制：统一动作、造数与 CLI-aware RL

**观测与动作。** 部分可观测：截图 \(I_t\) + 上一步 CLI 的 stdout/stderr。可执行动作统一为 `bash(c, δ)`，\(c\) 既可以是直接 shell，也可以是包着 pyautogui 的 Python heredoc；另加 wait / terminate / answer。统一 grammar 消融：分工具（computer_use vs cli）SFT 仅 38.8%，统一 bash 到 **46.0%**（+7.2 pp）。

**三类轨迹。** GUI only 转格式；CLI only 用文档技能 + Claude Code harness 采成功；interleaved 双通道采样与「可改写段 → CLI → 成功回放」。RLVR：按 CUA-Gym 式合成任务 + 可执行 verifier；每任务在 GUI / CLI / 混合下各采 16 条，按成功率（近平看步数）标 \(b^\star\in\{0,1\}\)：CLI 是否有清晰执行优势。

**奖励。** \(R_{\mathrm{CLI}}=\mathbb{I}[\mathrm{Success}]\mathbb{I}[b(\tau)=b^\star]\)；\(R(\tau)=R_{\mathrm{acc}}+\lambda_{\mathrm{CLI}}R_{\mathrm{CLI}}\)（\(\lambda_{\mathrm{CLI}}=0.1\)）；CLI 步 shell 失败 \(r^{\mathrm{exec}}=-1\)，以 \(\lambda_{\mathrm{exec}}=0.3\) 加进该动作 token 的优势。意图：轨迹级管 **when**，步级管 **how reliably**。

## 🧪 关键证据

**主表（OSWorld）。** HybridCUA-9B：Acc 53.6 / Steps 14.0。同尺寸对照：GUI 支线 SFT+RL 到 50.4 / 22.1；裸 GUI+CLI 基座 18.4。相对 AutoGLM-OS-9B +4.7、ToolCUA-8B +6.8、更大的 UltraCUA-32B +9.9，且无需 per-app API。

**SFT 消融。** 混合三类 46.0；仅 GUI 43.2、仅 hybrid 41.0、仅 CLI 31.7——单接口与混合轨迹互补，不是「全上 interleaved」就够。

**RL 消融。** 去掉 \(R_{\mathrm{CLI}}\)：准确率差不多，但步数只相对 SFT 短 18.2%（完整 29.3%），CLI 步占比停在 58.9% vs 64.0%。去掉 \(r^{\mathrm{exec}}\)：分与步尚可，执行错误回升到约 16.5%（完整约 11.5%）。两信号分管效率路由与命令可靠。

**OOD。** OSWorld-MCP 47.1（vs 基座 38.0，贴近 ToolCUA-8B 的 46.8）；WindowsAgentArena 36.0（+4.0 / 相对 ToolCUA +2.2）。域内行为：OS 任务 CLI 主导（约 84%），Chrome GUI 主导（约 74%）；内容编辑 / 结果核验偏 CLI，空间布局偏 GUI。

**泼冷水。** 收益依赖 shell 可用性与稳定性；Linux 训练迁 Windows 靠「路由」不等于命令知识完备；3K RL 任务的 \(b^\star\) 来自有限 rollout 排名，标签噪声可能；相对最大 EvoCUA-32B（56.7 GUI-only）仍有尺寸差距——作者主打同尺寸与「可扩展接口」叙事。

## 🔬 最有意思的部分

1. **暴露能力 ≠ 获得能力。** 裸加 CLI 准确率腰斩式下跌——评测若只报「支持终端」会系统性误导产品决策。
2. **统一 bash 语法是隐藏杠杆。** +7.2 pp 说明「两个工具名」会割裂策略学习；harness 动作合同值得认真设计。
3. **\(b^\star\) 让接口选择可验证。** 不是「多用 CLI 就加分」，而是「该用时用、不该用时别用」——与「最大化工具调用」类奖励划清界限。
4. **跨 OS 迁的是路由元技能。** PowerShell 出现而训练只见 bash，说明策略学到的是委派时机。
5. **与 coding-agent 工具链合流。** 文中明确对照 Claude Code / Codex / OpenClaw：CUA 下一步不是另起炉灶，而是把 shell 编排从「程序员默认」变成「视觉 agent 可学默认」。

## 🤔 我的判断

定位：**GUI–CLI 混合 computer-use 的数据 + CLI-aware RL 框架**（方法论文，附可发布资产承诺）。亮点三：

1. 用对照实验证明「给 shell 会伤分」，再证明训练后同接口变资产；
2. when/how 两级奖励拆得干净，消融可读；
3. 统一动作语法与跨 MCP / Windows 迁移，服务「少焊应用 API」的工程路线。

局限：标签与环境依赖；相对更大纯 GUI 专家仍有差距；代码/主页链接在 preprint 中以将释出为主。对 Byron：**(1)** coding-agent / CUA 沙箱默认同时挂 GUI 与 shell 时，必须有**接口路由评测切片**（该用 CLI 的任务 vs 不该用），别只看总分；**(2)** 动作 schema 优先统一可执行面（一个 bash / 一个 runner），工具名分裂会吃掉策略；**(3)** skills 可先服务「造 CLI only 轨迹」的文档蒸馏，再进在线策略；**(4)** judge/evals：步级 shell 失败信号几乎免费，应进 harness 信用分配，而不是只等任务末 \(R_{\mathrm{acc}}\)。接 [RecreationWorld](../202609/week4/20260922_RecreationWorld_论文解读_对着跑着的参考应用重写一遍_混合CUA才算真干活.md)（混合 CUA 验真干活）与本周 Raven（多 harness 编排）：HybridCUA 补的是**单工作站上 GUI 与 CLI 的可学习编排**。

一句话收束：**终端不是插件开关——HybridCUA 用混合轨迹与 CLI-aware 奖励，把「何时敲命令」练成和点击同等重要的策略。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
