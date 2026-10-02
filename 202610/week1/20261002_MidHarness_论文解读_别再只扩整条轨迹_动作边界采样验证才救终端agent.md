---
month: 202610
week: 1
date: 2026-10-02
type: 论文解读
slug: MidHarness
---

# 别再只扩整条轨迹：动作边界上的采样+验证，才把终端 agent 从「错命令毁环境」里捞回来

你有没有过这种体验：同一个模型、同一个 harness，跑 Terminal 任务时有时一把过、有时先 `npm test` 在没 `package.json` 的仓里炸开——环境已被改坏，后面再聪明也是在残骸上补丁。NVIDIA / KAIST 这篇 **Mid-Harness**（arXiv:2609.39982）把问题钉死在 **model–harness 交界**：生成出有用动作，不等于可靠执行；测试时算力该不该砸在「执行前」而不是只砸「整轨重跑」。看完我的感受：这是对 Byron 的 **control-plane · sandbox · LLM-as-judge** 一条极干净的切入——**生成器与 harness 都不动**，只在中间加一层 sample+verify，用 verifier 能力决定「多采样」有没有意义。

## 核心摘要

Mid-Harness 在每一步对同一交互历史采样 \(N\) 个候选动作，经 verifier 选中一个再交给未改动的 harness 执行；其余候选丢弃。主设定固定 **TMAX-9B**（Qwen3.5-9B + 终端 RL）与 Vanillux2 harness，评 **TerminalBench-Lite**。核心反直觉：**弱 verifier 下加宽采样几乎白费**（零样本 listwise \(N=4\to8\)，Pass@1 仅 49.32%→51.02%）；换 **GPT-5.6 Sol** 作前沿 verifier，同样 listwise 把 Pass@1 从基线 **50.00% 拉到 68.03%**（\(N=8\)），证明候选里已有可用替代动作。同模型自验证时 **pairwise 最优**（54.76% / Pass@3 71.43%），用 117k 条 Sol 两两偏好 **LoRA 蒸馏 verifier**（生成器冻结）再抬到 **57.14% / 75.51%**。与轨迹缩放组合：Best-of-\(T=3\) 单独 55.10%，加上蒸馏 Mid-Harness 到 **66.33%**（同三环境跑）；SR（\(R=1\)）+ Mid-Harness 到 **60.20%**，且可在低于 Best-of-\(T=7\) 的参考 token 成本下超过其成功率。转移：Terminal-Bench 2.1、SWE-bench-Verified Mini、FeatureBench-Mini，以及 Terminus-2 / Nemotron 等模型–harness 组合上，零样本 Mid-Harness 普遍抬 Pass@3、多数抬 Pass@1。

## 论文信息

- **标题**：Mid-Harness: Scaling Actions Between Model and Harness for Terminal Agents
- **作者 / 机构**：Minki Kang（NVIDIA / KAIST）等；Byung-Kwan Lee（Project Lead）；NVIDIA
- **仓库 / 项目页**：论文注明 project page（文内 link 占位）；方法以 model-call wrapper 伪代码描述
- **链接**：https://arxiv.org/abs/2609.39982 · https://arxiv.org/pdf/2609.39982
- **观察时间**：2026-10-02（HF Daily Papers / 终端 agent · test-time compute · action scaling）

---

## 🎯 为什么这件事值得写

终端 agent 的失败常常不是「不会想」，而是**已执行的坏动作改写了状态**：错装包、错改文件、错重试——后续观察全被污染。业界习惯把测试时算力花在 Best-of-\(T\) 整轨或 Sequential Refine 上，那都要额外环境实例与状态序列化。Mid-Harness 问的是另一轴：**在动作进 harness 之前**多花一点算力，能否用**单环境**换更高轨迹成功率？它还把「采样宽度」与「验证能力」拆开测——这正好对照 Byron 关心的 judge/evals：没有强 verifier，多采样只是多噪声。

## 🏗️ 机制：不动两端，只夹一层 Verify

形式化：历史 \(h_t\) 下从生成器 \(\pi\) 采 \(\mathcal{A}_t=\{a_t^1,\ldots,a_t^N\}\)，\(a_t^\star=\mathrm{Verify}_\psi(h_t,\mathcal{A}_t)\)，harness 只执行 \(a_t^\star\)。Verifier **看不到**候选的 `<think>` 推理，只看任务、观测历史与候选命令文本——逼它做「未执行前的效果判断」。

三种机制对照：（1）**Listwise**：一次看全集选一个；（2）**Pointwise**：逐候选打分再取最大；（3）**Pairwise**：两两比较（可配 tournament），\(N=8\) 时同模型下最强。集成点在 **model-call wrapper**：`generate` 改成 `n=N` + `verify`，外层 harness loop（parse / execute / update）零改动——这就是「Mid」：夹在 model 与 harness 中间的 control-plane 薄层。

蒸馏：从 244 个困难 TMAX-15k 任务的 732 条轨迹收 **117k** Sol 两两响应，LoRA 只挂在 verifier 路径；生成器仍是 Base TMAX-9B。

## 🧪 关键证据

**覆盖 vs 验证。** 前沿 Sol 把机会上界钉在 ~68% Pass@1；零样本 listwise 几乎贴基线——**「有候选」≠「会选」**。Pairwise 零样本 54.76%，蒸馏 57.14%；离 68% 仍有大缺口，但方向正确。

**成本。** \(N=8\) 时零样本 pairwise 的 verifier 总输出约 **125.7k** tokens/run，listwise 仅 **0.6k**——机制强也贵。有趣发现：部分设定下 **decision-only**（只吐 A/B）比带长推理的 pairwise 更省钱甚至更高 Pass@1（9B：零样本 / 蒸馏分别约 −20.9% / −24.1% 参考价成本且 Pass@1 升）。

**组合轨迹缩放（Table 2，9B）。** 基线 50.00；Best-of-3 55.10；零样本 Mid 54.76；蒸馏 Mid 57.14；蒸馏 Mid+Best-of-3 **66.33**；蒸馏 Mid+SR **60.20**。Figure 5：蒸馏 Mid@\(N=8\) 约以 Best-of-5 的 **三分之一**参考成本打到同档 57.14%；再与 SR/BoT 组合可超过 BoT@\(T=7\)（59.18%）且更便宜。

**转移（Table 3）。** Terminal-Bench 2.1 上 TMAX-9B：21.72→27.34（零样本）；FeatureBench-Mini 基线仅 1.45%，零样本 / 蒸馏到 5.80 / 7.25；SWE-Verified Mini 亦有小幅增益。Terminus-2 上的 Qwen3.5-9B / Nemotron 同样受益。

**泼冷水。** 蒸馏后离线与教师的 verification agreement 仅到 **57.79%**（pairwise 偏好一致 74.58%）；剩余分歧里 **命令语义 + 执行可行性**约占清晰失败的 **67.4%**——正是「不执行就判断命令在当前环境会干什么」的硬核。4B 上单独 Best-of-3 甚至掉分（−4.09 pp），说明轨迹 verifier 也要配模型能力。

## 🔬 最有意思的部分

1. **弱 verifier ≠ 多采样。** 把「test-time compute」从口号拆成必要条件：候选覆盖 × 可验证性。
2. **动作缩放与轨迹缩放互补，且省环境。** Mid-Harness 不增加 #Env；BoT/SR 要 2–3 份环境——对难序列化的真实终端更友好。
3. **坏动作的不可逆性**是动机核心：验证的价值不在「多想一句」，而在**阻止污染后续观测**。
4. **Decision-only** 暗示：动作级偏好有时不需要长 CoT——对 harness 里塞廉价 gate 很有工程暗示。
5. **生成器冻结、只训 verifier**：和「烤 harness 进权重」相反——控制面可独立升级。

## 🤔 我的判断

定位：**终端 agent 的动作级 test-time 算力轴 + 验证机制系统研究**（方法薄、测量厚）。亮点三：

1. 把 model–harness 边界做成可插拔 Mid 层，两端零改；
2. 用前沿 verifier 标定「候选覆盖上界」，再诚实展示自验证与蒸馏的回收比例；
3. 证明动作缩放可与 BoT/SR 叠加并改善成本–成功率前沿。

局限：主数字绑 TMAX + TerminalBench-Lite；Sol 作 oracle 贵；pairwise 成本陡；蒸馏仍卡在命令语义/可行性。对 Byron：**(1)** MMP / Pi 的 control-plane 可抄「generate wrapper 里 sample+verify」，ambient 仍关、Manifest 只加一层 Mid；**(2)** 弱 judge 时别盲目加宽 \(N\)——先校准 verifier（对齐上周 [LLM-as-Judge](../202609/week3/20260917_LLM-as-Judge_论文解读_裁判只是评测栈的一层如何搭闭环.md) 的「代码检查先于 LLM 裁判」）；**(3)** 与 [HybridCUA](./20261001_HybridCUA_论文解读_光给shell不够_学会何时用CLI才不在OSWorld掉分.md) 对照：那边学「何时用 CLI」，这边学「执行前挡坏命令」——都是 sandbox 里的策略面，不是更大基座；**(4)** 接 [Meta-Harness](../202609/week3/20260918_Meta-Harness_论文解读_脚手架也能端到端搜_完整轨迹文件系统比压缩反馈狠.md)：搜整个 harness 贵，Mid-Harness 先把最贵的不可逆动作闸住。

一句话收束：**终端失败常毁在已执行的坏动作——Mid-Harness 证明：在 model 与 harness 之间缩放「采样+验证」，比只加宽整轨重跑更对症，且强 verifier 才解锁候选里的替代动作。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
