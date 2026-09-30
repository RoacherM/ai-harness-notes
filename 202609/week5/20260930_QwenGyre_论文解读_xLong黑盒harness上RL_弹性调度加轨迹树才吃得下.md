---
month: 202609
week: 5
date: 2026-09-30
type: 论文解读
slug: QwenGyre
---

# xLong 黑盒 harness 上 RL：弹性调度 + 轨迹树，才吃得下百万 token rollout

你有没有过这种体验：想在 Claude Code / Codex 这类**部署同款黑盒 harness**上做 online RL，一次任务跑数小时、近百万 token、上百次模型–环境交互——Colocate 等长尾 rollout 把 GPU 晾着，Async 死分区又让训练侧空转；更糟的是 compaction / 子代理把历史撕成**非线性轨迹图**，展开训练样本前缀爆炸。阿里 Token Hub 这篇 **QwenGyre**（arXiv:2609.33848）正面拆这两难：**弹性调度**在不停活执行的前提下重切 rollout/训练 GPU；**轨迹处理器**用 TITO + 轨迹树保原上下文、超时可评部分分、按角色优先级限幅采样并让共享前缀只计一次。看完我的感受：这正好接在 Byron 的 **harness / coding-agent RL** 线上——不是再发明一个玩具 agent loop，而是承认「工业 harness 不可重写进 RL 框架」，用代理层吃黑盒。

## 核心摘要

QwenGyre 面向 xLong-horizon online RL：单次执行可持续数小时、数百次交互、约 1M token。**弹性调度**把 GPU 编成可切换 cell（core 持权威权重与优化器；satellite 可中途加入正在进行的 batch），按 waterlevel（未完成执行量）在服务活 harness 的同时把空闲容量拨去训练，并配合 streaming micro-step 与动态数据并行。**轨迹处理器**在黑盒代理上做 token-in/token-out 记录，建成共享前缀轨迹树；终止后对保留工作区评分（含超时后可评估的部分进度）；每执行最多录取 \(J_{\max}\) 条轨迹（主代理 > 主摘要 > 子代理 > 子摘要），掩码非策略 token，共享目标只训一次，损失先在执行内对可训 token 平均再对 batch 平均。旗舰 **Qwen 3.8 2.4T**、约 700K token/rollout 上 NL2RepoBench 48 步 **52.5%→58.5%**；相对 Colocate / Async 端到端加速最高约 **1.85× / 1.78×**，训练分曲线可比。亦在 DeepSWE、TerminalBench 与 Qwen 3.6 122B 上验证。

## 论文信息

- **标题**：QwenGyre: An Elastic Reinforcement Learning Framework for Training xLong-Horizon Agents
- **作者**：Weiqi Wang*、Yuxin Zhou*、Mouxiang Chen*、Siyuan Zhang、Yi Zhang、Yuyan Luo、Zhiyu Yin、Chencan Wu、Jiemin Jiang、Wentao Yao、Chujie Zheng、JianWei Zhang†（* 共一；† 通讯）
- **机构**：Alibaba Token Hub, Alibaba Group；中科大；清华；浙大（部分作者）
- **链接**：https://arxiv.org/abs/2609.33848 · https://arxiv.org/pdf/2609.33848
- **观察时间**：2026-09-30（HF Daily Papers Sep 29；xLong coding-agent RL / harness-native 训练）

---

## 🎯 为什么这件事值得写

Coding agent 的 RL 叙事已从「短工具环」走到 DeepSWE / NL2Repo 级长程，但系统论文常默认**自研线性 agent loop**。真实部署依赖会 compaction、开子代理、快速迭代的闭源/半闭源 harness——**不能也不该**为了 RL 重写控制流。QwenGyre 的贡献在「调度平面 × 数据平面」同时对准 xLong：**活执行不能因切 GPU 角色而断推理**；**分支轨迹不能扁平平均成噪声**。对 Byron / MMP：这是「Manifest 钉死的工业 harness」与「组相对 RL」之间缺的那层 runtime。

## 🏗️ 机制：弹性调度与轨迹处理

**组相对管线。** 每 query \(N\) 条独立 rollout 成组算优势；每步消费 \(B\) 组；burst 内 \(\mu\) 步再发布权重。调度 staleness \(d=v_t-v_d\) 与边界派发深度 \(Q_b=(\varphi+\mu)B\) 有解析关系。

**弹性调度。** \(K\) 个 cell 共享训练并行布局；可选独立 rollout 池。Waterlevel \(w=d_{\mathrm{dispatch}}-f_{\mathrm{finish}}\) 下降且供给就绪时，按序把 cell 切入训练（Eq. 覆盖「剩余 rollout 容量 ≥ 未完成工作」）。角色切换时代理重路由模型请求、可选 RDMA 迁 KV，**harness/沙箱状态留在 GPU 外**。Streaming：轨迹缓冲 → 按需打包 micro-step → cell 争抢队列，使参与训练的 cell 近似同时收工；satellite 可在 batch 中途 pull 参数加入。

**轨迹处理器。** TITO 记录精确 token 与行为 logprob；工具调用 payload 按 ID 缓存以抗 harness 重格式化。轨迹树共享前缀、分支于上下文分叉。部分评分：超时后仍可对已提交产物打有效分；评估失败≠零分。采样：按角色优先级填满 \(J_{\max}\)（默认 5），路径一旦录取即掩掉树上共享前缀，防重复计权。

## 🧪 关键证据

**NL2RepoBench × Qwen 3.6 122B。** 单执行均长约 1.93h、query 约 2.96h；48 步相对 Async 约 1.42–1.53×、相对 Colocate 约 1.36–1.47×，训练分对齐。角色时间线显示 cell 错峰进训练，切换开销秒级相对小时级 rollout 可忽略。

**旗舰 Qwen 3.8 2.4T。** 更重超时：完整时长记录里 61.1% query ≥4h。QwenGyre 48 步 75.4h vs Async 134.6h（1.78×）、Colocate 91.5h（1.21×）；评测 passrate 52.48→58.54。超时扎堆时「早回收 cell」空间变小，故相对 Colocate 增益收窄——作者解释与调度直觉一致。

**DeepSWE / TerminalBench。** 24 步配置下 vs Async 约 1.38–1.57×、vs Colocate 约 1.61–1.85×。

**消融。** Streaming  alone 不够（Async 死分区仍在）；Colocate+standalone 仍粗粒度；细粒度 cell+streaming（E0）才到 1.0 基准时间。\(J_{\max}=1\)（仅主轨迹）训练分更差、梯度范数更大；\(J_{\max}=5\) 逼近不封顶采样且前向–反向时间约为不封顶的 74.8%。

**泼冷水。** 需要足够多可独立切换的 cell，极小 GPU 预算不适用；streaming+多步 burst 无法做全局 shuffle；学习质量对 sample 顺序的影响未单独隔离；黑盒接口依赖代理正确性与 harness 协议稳定性。

## 🔬 最有意思的部分

1. **三个生命周期解耦。** Harness 执行、GPU 角色、训练样本——切角色不断活任务，是 xLong 能训的前提。
2. **部分分是一等公民。** 超时不等于「全零失败」；可评估的部分进度保留优势差异——否则长尾全被同一失败奖励糊掉。
3. **执行级损失归一。** 防「分支多的执行霸占 batch」——对子代理狂开的 harness 尤其关键。
4. **与 ClawGym II / LEGO-RL / Polar 同族。** 都在吃黑盒；QwenGyre 把弹性 actor 重配与带角色/分支/评估出处的轨迹处理绑成端到端。
5. **调度论文写进 agent 叙事。** Waterlevel、staleness 公式不是附录装饰——没有它们，「在 Claude Code 上 RL」会停在 demo。

## 🤔 我的判断

定位：**面向工业黑盒 harness 的 xLong agentic RL 系统**（调度 + 轨迹语义，附旗舰模型验证）。亮点三：

1. 承认不可重写 harness，用代理与 TITO 保住训练语义；
2. 弹性 cell 同时打穿 Colocate 尾部空转与 Async 死分区；
3. 轨迹树 + 角色限幅 + 执行级平均，把分支爆炸收成可算目标。

局限：资源下界、shuffle、跨 harness 协议演进。对 Byron：若 MMP / Pi 要接 RL——**(1)** 把「模型调用代理 + 轨迹树落盘」做成 Extensions，而不是改 Manifest 里的 agent loop；(2) 超时策略区分 harness timeout（仍评分）与 overall timeout（丢弃）；**(3)** 训练侧默认 \(J_{\max}\) 与主/子/摘要优先级，避免摘要轨迹冲淡任务轨迹；**(4)** 评测与门禁继续用可执行检查，部分分规则写进任务合同。接本周 [Gagar](./20260930_Gagar_论文解读_测试过了还不够_组内agentic裁判才把信用拨给可合并补丁.md)（组内质量信用）与 [WideSWE](./20260930_WideSWE_论文解读_单仓绿了不算完_跨仓协调才是coding_agent真考场.md)（跨仓完备）：QwenGyre 补的是**时间轴上的超长执行如何真正进得了 RL 循环**。

一句话收束：**xLong 不是把短环拉长——QwenGyre 用弹性 GPU 与轨迹树，让黑盒 harness 上的百万 token rollout 变成可训练、可加速的数据。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
