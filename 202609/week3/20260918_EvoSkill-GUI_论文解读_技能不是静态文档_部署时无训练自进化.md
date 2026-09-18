---
month: 202609
week: 3
date: 2026-09-18
type: 论文解读
slug: EvoSkill-GUI
---

# 技能不是静态文档：EvoSkill-GUI 让 GUI agent 在部署时无训练地改包、复用

你有没有过这种体验：给 GUI agent 写好一份「技能说明」，弹窗一出、控件一挪、a11y 树一旧，整条长程计划当场作废；失败轨迹里明明写着「该换备用定位 / 缺恢复分支」，但这些教训锁在单次 rollout 里，下一题还从零编。浙大 ZJU-REAL 这篇 *Reflect, Revise, Reuse: Training-Free Skill Evolution for GUI Agents*（EvoSkill-GUI，arXiv:2609.17653）主张：GUI 需要的不是更好的**静态** skill，而是能在部署期根据执行反馈**改写自身**、且**无需训练**的程序知识。看完我的感受是：它和 Byron 的 Wayne-Skills / ComposeCL 对话很直接——ComposeCL 讲权重侧持续学习组合，EvoSkill 讲 **skill 包文件系统侧的 inference-time 进化**，信息隔离 critic 又碰到 judge 栈。

## 核心摘要

EvoSkill-GUI 把每个 skill 做成结构化多文件包 \(S=(D,A,P,B,C,F)\)：检索元数据 \(D\)、无障碍工具 \(A\)、可执行计划 \(P\)、备用定位 \(B\)、失败恢复 \(C\)、失败案例集 \(F\)。运行时走 **reflect–revise–reuse**：executor 可在 rollout 内对矛盾观察做 instant revision；失败后**同一 backbone** 另开 session 当 critic，信息集严格 ⊆ 指令+观察+动作（**看不到 skill 正文、executor CoT、GT**），再经受限工具接口（`read/write/append/list/search/create_failure`）改对应文件。库用元数据检索（intent/app/platform/kw…），阈值 \(\theta_r=0.6\) 默认。MobileWorld / AndroidWorld / OSWorld 上，对多类基座（Claude-Sonnet-4.6、Qwen3.6、GUI-Owl、MAI-UI 等）**零训练**提升，最大约 **+16.2 / +6.0 / +10.5** 个百分点；三轮进化贡献主要增益后趋于饱和；相对 pass@3 重复采样，三轮进化用略少 token 拿到更高成功率。AndroidWorld 116+116 流式任务上，库从个位数涨到近百，Phase 2 复用率 **100%** 仍有 +6.0 点——说明在攒可迁移程序经验，而非每题一次性脚本。

## 论文信息

- **标题**：Reflect, Revise, Reuse: Training-Free Skill Evolution for GUI Agents
- **作者**：Bofan Chen*, Boxuan Zhang*, Fei Tang, Zhengxi Lu, Yong Du, Tongbo Chen, Weiming Lu, Jun Xiao, Yueting Zhuang, Yongliang Shen†（*共同一作）
- **机构**：ZJU-REAL, Zhejiang University
- **链接**：https://arxiv.org/abs/2609.17653 （2026-09-15）· 代码 https://github.com/ZJU-REAL/EvoSkill-GUI · 项目页 https://zju-real.github.io/EvoSkill-GUI/

---

## 🎯 现有 skill 范式在 GUI 上卡死的四件事

长程 GUI 是非平稳 POMDP：截图 \(\varphi_t\) + a11y 树 \(\zeta_t\) 构成观测，弹窗/延迟加载/控件迁移会让「执行前写死的 plan」失效。作者归纳现有 skill 框架的四道坎：

1. **非结构化单文件**——计划、定位、恢复缠在一起，难定点修改；
2. **界面非平稳**——幂等失败环吃掉大量超时；
3. **外部 VLM 反思**——额外延迟，且与执行/改包弱耦合；
4. **程序知识不累积**——失败教训不进资产库。

并发的自进化 skill 多偏代码环境，或只做检索式记忆更新而不改 skill 本体。EvoSkill 的主张是：失败本身携带「改哪一部分」的信号——grounding 错 → 改 \(B\)，缺 contingency → 改 \(C\)，流程错 → 改 \(P\)——应**立刻写回包**，而不是只留给下次训练。

## 🏗️ 机制：结构化包 + 隔离反思 + 元数据复用

### 包长什么样

```text
skill_package/
|-- meta_info.json      # D: intent, app, platform, keywords, args, hist, status
|-- docs/
|   |-- plan.md         # P: 做什么
|   |-- backup.md       # B: 换一种怎么找
|   `-- recover.md      # C: 打断与恢复
|-- a11y_utils/         # A
`-- failure_examples/   # F
```

目标函数形式化为在任务分布上最大化期望完成回报；优化手段不是梯度，而是对 \(S\) 的受限编辑。

### Reflect–Revise 三拍

1. **Rollout + instant revision**：执行中可用 \(\mathcal{T}\) 做局部改包，拦住「控件挪了」这类级联失败。
2. **Isolated critic**：同参异 session；\( \mathcal{C}_J \subseteq \{I\}\cup o_{1:T}\cup a_{1:T} \)，与 \(\{S,\mathrm{CoT},GT\}\) 交为空——防止「偷看答案式」伪进化。
3. **Revision**：按 critique 改文件；失败则 `create_failure` 追加证据案例。成功或达到轮次上限停止（实验默认失败后最多 **2** 轮修订，交互步 ≤50）。

### 检索

不对长正文做糊匹配，而用元数据打分：

\[
\mathrm{base}=0.6\frac{|O|}{|Q|}+0.4\frac{|O|}{|Q\cup T_S|},\quad
\mathrm{score}=\mathrm{clip}(\mathrm{base}+b_{\mathrm{app}}+b_{\mathrm{kw}}-p_{\mathrm{div}},0,1)
\]

超 \(\theta_r\) 复用，否则新建；仅成功或经自进化后的包入库。阈值过低（0.4）SR 掉到 55.2%，过高（0.8）到 66.7%，**0.6 → 69.5%**（MobileWorld）。

## 🧪 关键证据：三榜 + 消融 + 复用流

**MobileWorld（GUI-only）**：Claude-Sonnet-4.6 57.1%→67.6%；Qwen3.6-Plus 53.3%→69.5%（最大约 +16.2）；开源 Qwen3.6-35B-A3B 32.4%→44.8%；MAI-UI-8B 29.5%→37.1%。跨闭源/开源、通用/GUI 特化一致受益。

**AndroidWorld**：Phase1（seed30）reuse 37.9%，68.1%→70.7%（+2.6）；Phase2（相关 seed42）reuse 100%，55.2%→61.2%（+6.0）。失败→成功恢复 27/89（含复用技能 19/65）。

**OSWorld-Verified**（≤50 step）：GUI-Owl-1.5-8B 46.7%→54.8%（+8.1），VLC +42.7、Thunderbird +20.0；Qwen3-VL-8B 23.8%→34.3%（+10.5）。附录诚实讨论少数负域（如某次 OS 域 −4.1）：可能是短直达任务被 skill 过度复杂化，且小域对单题波动敏感。

**消融**：去 plan.md 掉 3.81 点（最大）；去 failure reflection 掉 2.85；加 a11y 对多 backbone +1.9–3.8。修复性分析：plan 级错误 16/23 可修，grounding 6/12，缺 contingency 仅 1/4——**结构分文件不是美学，是可修复性设计**。

**成本**：三轮 EvoSkill（Qwen3.6-Plus）69.5% / 94.75M tok vs pass@3 基线 62.8% / 103M tok——增益不是「多采样刷出来的」。轮次曲线：前三轮吃掉大部分涨幅，第 4–5 轮几乎饱和。

## 🔬 最有意思的部分：single-backbone 自进化与「可审计改包」

两点对工程特别狠：

1. **不引入更强外部裁判**——critic = 同模异 session + 信息隔离。这卡死了「用 GPT-裁判给小模刷分」的偷懒路径，也逼着你把 judge 协议写清楚（和 Byron 的 LLM-as-Judge / ACP 笔记同题）。
2. **改包走受限工具而非自由改仓库**——修订局部、可审计；失败案例成为一等资产。这和「skills 应版本化、可 diff」的 Wayne-Skills 直觉一致，只是进化发生在**部署环**而非人工 PR。

局限同样清楚：感知/诊断不稳时可能写入有害修订，且**没有形式化 veto**；真实部署的个性化配置、网络延迟、隐私态远超三榜；AndroidWorld 高复用率建立在同任务族参数化变体上，跨应用品类迁移会更陡；每失败轨迹多一次 critic + 一次 revision 的推理税，需靠复用摊销。

## 🤔 我的判断

定位：**GUI skill 的 inference-time 自进化方法 + 可落地的包格式规范**。亮点三：

1. 把「静态 skill 文档」升级为可定点编辑的多文件包，并与失败类型对齐；
2. 信息隔离 critic 把「自进化」从口号收成可辩伪协议；
3. 三榜多模一致增益，且用 pass@3 对照证明不是刷采样。

对 Byron 栈：

- **Wayne-Skills**：包目录 schema（plan / backup / recover / failure_examples / meta）可直接当 skill 规范草案；检索应走元数据而非全文糊配。
- **judge / ACP**：隔离 critic 的信息集定义值得抄进评测栈——防止裁判偷看 skill 或 GT。
- **Pi / DSH / CL**：EvoSkill 是环境侧「活程序记忆」；与 SoL-Pi 的效率机制、Harness-Design 的条件化脚手架正交——一个管 **GUI 程序知识怎么长**，一个管 **coding harness 怎么省/怎么配**。
- **ComposeCL**：权重持续学习 vs 包持续修订——长期应回答「哪些应进权重、哪些应留在可 diff 的 skill 文件」。

一句话收束：**部署期最贵的不是再训一版 VLM，而是把已经付出的失败轨迹，写成下一题还能检索到的、可审计的程序补丁。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
