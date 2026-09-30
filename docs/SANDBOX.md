# 沙箱：各家怎么实现，以及我们的沙箱 SDK

> 回答你的两点：**① 主流项目（Codex / Claude Code / Deep Agents / dsh / gemini-cli / letta / 云端沙箱厂商）到底怎么实现沙箱；② 我们这边要一个能 `import` 的沙箱 SDK，方便执行沙箱——它长什么样。**
> 配套：[ARCHITECTURE.md](./ARCHITECTURE.md) §3 seam 表、[BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md)（`bash` 工具与能力探测）、[HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md)（四道闸）
> 状态：**方案稿**。§8 有 4 个需要你拍板的点；§7.3 有一条关于**你这台 Windows 开发机**的现实结论，建议先看那条。

---

## 0. 先把两条信息掰清楚（不掰会一路错到底）

### 0.1 "沙箱"这个词在业界指两种**正交**的隔离

我这次调研最大的收获不是"谁用了什么技术"，而是发现七家说的"沙箱"根本不是同一件事，而且大多数宣传刻意混着说：

| 语义 | 它防什么 | 它管什么 | 代表 |
|---|---|---|---|
| **A. 执行隔离**（系统调用 / 进程 / 容器 / microVM） | **agent 逃逸到系统** | 子进程能调哪些 syscall、能读写哪些路径、能连哪些地址 | Codex、gemini-cli、Claude Code、dsh、letta-code |
| **B. 工作区视图隔离**（写时复制副本） | **agent 改坏仓库** | "改动落在哪一份副本上"，产物 diff 回主线；**不约束 syscall** | oh-my-pi 的 `pi-iso`（8 种后端）、我们已设计的 `checkpoints` / `teams.worktree` |

**两者正交，理想是叠加。** 反例：pi-iso 给了子 agent 一个工作区 CoW 副本，但它在副本里**照样能任意执行命令、照样能联网**——它只保证"改不坏你的仓库"。而 Codex 的 Seatbelt 能挡住 `/Users/you/.ssh`，但不懂"这是哪个分支的副本"。

**我们的裁决（本文的主线）**：
- `ctx.sandbox` = **A 类**，管"能不能碰"。
- `ctx.checkpoints` + `teams.worktree`（v2.1 已定）= **B 类**，管"落在哪里"。
- 两个 seam 独立存在、可以叠加，**绝不合并成一个叫 `sandbox` 的东西**。文档和 `SECURITY.md` 里必须逐条写清"隔离的是哪一层"——opencode 的 `SECURITY.md` 是范本：它直接声明"权限系统是 UX 特性，不提供安全隔离，要真隔离就去 Docker/VM"。**明确写出"不提供什么"和写出"提供什么"同样重要。**

### 0.2 "内置了沙箱" ≠ "默认在沙箱里"

业界现状比宣传难听得：`smolagents` / `AutoGen` / LangChain / OpenAI Agents SDK 的**零配置默认都是本机直接执行**。Claude Agent SDK 的 `sandbox.enabled` 默认 `false`；letta-code 的 agent 自身 shell 默认**不**沙箱（要 `LETTA_FS_SANDBOX=1` 才开）；smolagents 的 `LocalPythonExecutor` 在自己 docstring 里写死了 *"It is not a security sandbox"*。真正默认关进笼子的只有 AgentScope、OpenClaw、OpenHands 的容器部署形态，以及 Codex。

由此定我们的一条铁律：**默认值就是安全边界的一部分，且必须可验证。** 我们的 `agentic doctor` 必须打印"当前实际生效的隔离等级 + 它是默认值还是被谁 patch 的"，`session log` 里每个 `bash` 调用都要记下它跑在哪个笼子里（见 §3.6）。

---

## 1. 技术谱系：从轻到重六档（决定"我们做到第几档"）

| 档 | 机制 | 隔离了什么 | 启动 | 要特权/daemon | 谁在用 |
|---|---|---|---|---|---|
| 1 | **解释器级**（AST 解析 + import 白名单） | 模块与内置函数面 | 极快 | 否 | smolagents Local（**非安全边界**） |
| 2 | **进程级降权**（低权限用户 / 受限令牌 / ACL） | OS 权限面 | 极快 | 视实现 | Codex Windows 受限令牌、dsh Win ACL |
| 3 | **系统调用过滤**（seccomp / Landlock / Seatbelt） | 进程能触达的内核能力面 | 快 | 需内核支持 | Codex 三平台、Claude Code、qwen-code |
| 4 | **文件系统视图**（bubblewrap mount namespace） | 文件系统视图 | 毫秒 | **否**（user ns） | Codex Linux、letta-code、dsh |
| 5 | **容器**（namespace + cgroup + overlayfs + seccomp） | 视图 + 资源 + 文件层 + syscall | 秒级 | 需 daemon | gemini-cli、qwen-code、OpenClaw、AgentScope |
| 6 | **microVM**（Firecracker / gVisor / Kata） | 独立内核 | 百毫秒级 | 需宿主管理面 | E2B、Daytona、Modal、Vercel |

三个必须记住的取舍：
- **第 3 档配错会误伤正常 syscall**，程序以诡异方式崩。生产做法是"先跑探针再收紧"（我们 `probe()` 就是这个探针，见 §4.5）。
- **第 4 档 bwrap 只隔文件系统视图**，网络 / 进程 / 资源仍共享宿主。别把"有 bwrap"当完整沙箱。
- **第 5 档容器共享宿主内核**，内核漏洞可逃逸；但它顺带解决了**资源限制**（`--memory` / `--cpus`），这是我们预算层需要的。
- **越轻的档越要叠加**：Codex = 3+4 叠加 + 独立网络代理；letta = 纯 4；OpenClaw = 纯 5。

---

## 2. 各家具体怎么做的

### 2.1 Codex：三平台**各用一套 OS 原生机制**，不走容器

`codex-rs/sandboxing/src/manager.rs`（约 36–76 行）按平台三选一：

| 平台 | 机制 | 落点文件 |
|---|---|---|
| macOS | **Seatbelt**（`sandbox-exec` + SBPL 策略） | `sandboxing/src/seatbelt.rs` 内嵌 `seatbelt_base_policy.sbpl` / `seatbelt_network_policy.sbpl`，200+ 行规则，含 **Mach IPC 白名单** |
| Linux | **bubblewrap + seccomp**（Landlock 已降级为 legacy） | `linux-sandbox/`；`landlock.rs` 347 行文件头注释自己写着"文件隔离已改由 `linux_run_main` 里的 bwrap 负责，本文件保留为 legacy 备用" |
| Windows | **受限令牌** + ACL | `windows.rs` / `windows-sandbox-rs`；WSL 沿用 Linux 那套（明确拒绝 WSL1，因为 bwrap 与 WSL1 的内核兼容层不兼容） |

三条关键判断：
1. **理念是"先关笼子，再放权"**，而不是每条命令弹窗问人。Claude Code 走的是审批路线（权限粒度做得很细），代价是**审批疲劳**——一晚上几百条命令，用户从第五条开始闭眼按回车，审批形同虚设。Codex 默认假设命令不可信，先丢进 OS 沙箱跑，炸也炸不出工作目录；**笼子不够用时才升级成问人**。
2. **Linux 的文件隔离换过一次血**（Landlock → bubblewrap 用户态），旧代码留在原地当化石。这种决策不读源码不知道，官方文档一个字没提。启示：**沙箱实现要允许换底座**，所以我们把 `SandboxBackend` 定成 interface 而不是把 SBPL/bwrap 参数写进 provider 上层。
3. **网络是唯一做了完整一层的**：独立 `network-proxy` crate，在沙箱内拦截子进程 HTTPS 流量做 **MITM**，按域名黑白名单放行；**凭据代理在转发时注入认证**（内置 GitHub / OpenAI 两个 provider），于是沙箱内进程**既不直接持有密钥、也不能随意联网**。域名策略**同时进 system prompt**（`<network>` 标签下列出允许与拒绝），模型知道边界才不会瞎试。
4. 失败姿态：**fail-closed**——沙箱起不来就报错，不静默降级。

### 2.2 Claude Code：沙箱化 Bash 工具 + 一个**独立可复用的沙箱运行时包**

对我们的 SDK 设计信息量最大的一家，因为它同时给了"配置面"和"可复用 npm 包"。

- **`@anthropic-ai/sandbox-runtime`**：把**整个进程**包进与内置 Bash 沙箱相同的 Seatbelt / bwrap 隔离里。macOS 零额外依赖；Linux/WSL2 需要 `bubblewrap` + `socat`（网络中继）+ `ripgrep`。配置放 `~/.srt-settings.json` 或 `--settings`。这是"沙箱 SDK 该怎么对外发布"的直接参照物。
- 内置沙箱**只覆盖 Bash 工具**：`Read`/`Edit`/`WebFetch` 在进程内跑、不走子进程，靠 permission rules 管；**MCP server 与 hooks 是宿主上的独立进程，完全不受 Bash 沙箱约束**。要全都关进去就得用上面那个 runtime。← 这条边界声明我们必须照抄进 `SECURITY.md`。
- 策略面（可直接借形状）：`sandbox.filesystem.{allowRead,denyRead,allowWrite,denyWrite}` + `sandbox.network.{allowedDomains, allowUnixSockets, allowAllUnixSockets, allowLocalBinding, httpProxyPort, socksProxyPort}` + `excludedCommands` + `failIfUnavailable`（默认 `false`＝**静默降级**，官方建议托管部署设 `true`）+ `autoAllowBashIfSandboxed`。
- **覆盖优先级算法定得很死**（值得抄，见 §4.3）：路径越窄的规则越优先；**`deny` 在更宽的 `allow` 内部仍然生效**，所以一个宽 `allowRead: ["~/"]` 不会把 `~/.env` 偷偷放回来。
- **逃生舱 `dangerouslyDisableSandbox`**：命令被沙箱挡下时，**违规信息写进 tool result 告诉模型是哪个路径/域名被拒**，模型可以带这个参数重试；重试走常规权限流程并弹窗。可用 `allowUnsandboxedCommands: false` 关掉（"Strict sandbox mode"）。
- 沙箱内自动放行的例外定得极细：显式 deny 规则永远生效、`rm` 打到关键路径仍走审批、内容级 ask 规则（`Bash(git push *)`）照旧弹窗、但裸 `Bash` ask 规则对沙箱内命令跳过。

### 2.3 gemini-cli（以及 Qoder CLI / qwen-code）：**多后端 + 平台自动探测**

- 六种后端，覆盖强度梯度：`docker` / `podman` / `sandbox-exec` / `runsc`(gVisor) / `lxc` / `windows-native`。
- `getSandboxCommand()` 按平台自动选：macOS 优先 `sandbox-exec`，否则 `docker` → `podman`（`sandboxConfig.ts:43-109`）。启用方式三级：`-s/--sandbox=<provider>` > `GEMINI_SANDBOX` 环境变量 > `settings.json` 的 `tools.sandbox`。
- **Seatbelt profile 做成 2×3 的矩阵**（qwen-code 六套）：`permissive` / `restrictive` × `open` / `closed` / `proxied`（网络三档）。这是"策略即配置"的形状，比一堆散 flag 好记好测。
- 容器路线支持项目自定义镜像：`.gemini/sandbox.Dockerfile`（`FROM gemini-cli-sandbox`）+ `BUILD_SANDBOX=1`；资源限制直接透传 `SANDBOX_FLAGS="--cpus=2 --memory=2g --network=none"`。
- SBPL 用**参数化路径**（`(subpath (param "TARGET_DIR"))`、`INCLUDE_DIR_0..4`）而不是硬编码字符串——这样同一份 profile 能跨环境复用，也方便我们做单测快照。
- 一个诚实的更正：**gemini-cli 没有硬编码屏蔽 `.env`**（网上大量文章说它有）。沙箱禁用路径来自用户的 `.geminiignore`。别把"我配了 ignore"当成"我提供了安全"。

### 2.4 Deep Agents：`Sandbox` 是**一种 Backend**，而不是外挂

这是我在 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §1 里已经拆过的框架，沙箱部分是同一套逻辑的延伸：

- `SandboxBackendProtocol` = `BackendProtocol`（七方法）**再加 `execute`**。实现了它，`execute` 工具才出现在模型可见面上；没实现就被**自动隐藏**（不是返回"不支持"错误）。
- 现成 provider：LangSmith / AgentCore / Daytona / Modal / E2B / Runloop / Vercel / OpenSandbox，`dcode --sandbox <provider>` 一行切；接入成本极低（一个 backend 包，导入时自动注册）。
- **`LocalShellBackend` 就是那个反面教材**：`subprocess.run(shell=True)`，**零隔离**，官方直接列风险（可读任意文件含 secrets、配合网络工具可外泄、修改不可逆、可吃满 CPU/内存/磁盘），并明确写 **"`virtual_mode=True` 在有 shell 权限时不提供任何安全性，因为命令能访问宿主任何路径"**。`timeout` 默认 120s、`max_output_bytes` 默认 100_000。
- 官方给的用法分层：开发期用 `FilesystemBackend`/`LocalShellBackend`，**生产环境一律换 sandbox backend**；用 `CompositeBackend` 把 `/proj/` 路由到本地、`/memories/` 路由到 StateBackend、`/sandbox/` 路由到 Daytona——**模型对切换无感知**。

**这里有一个我们要偏离它的地方，见 §4.1。**

### 2.5 letta-code：**纯路径策略 → bwrap 参数**，最短可抄的一段

TS 项目（`packages/...`），策略结构就五个字段，非常合我们的胃口：

```ts
export interface FsSandboxPolicy {
  baseWritableRoots: string[];   // 大范围基础可写
  deniedRoots: string[];         // 遮蔽（读不到也枚举不到）
  readonlyRoots: string[];       // 只读根
  writableRoots: string[];       // 最精确的可写 carve-out
  restrictWrites: boolean;       // 全局只读根视图
}
```

`buildBwrapArgs(policy)` 的生成顺序（**顺序即语义，不能调**）：
`--ro-bind / /`（或 `--bind`）→ `--dev /dev --proc /proc` → 对每个 `baseWritableRoots` 上 `--bind-try` → 对每个 `deniedRoots` 上 **`--tmpfs`（目录不是"不可写"，而是直接"缺席"）** → 再按 `readonlyRoots`/`writableRoots` 上 `--ro-bind-try`/`--bind-try` → 最后 **`--die-with-parent`**。

三条血泪细节，**必须进我们的实现与单测**：
1. **bwrap 是 last-mount-wins**：carve-out 必须是 `deniedRoot` 的**后代或不相交**，绝不能是**祖先**——祖先会把整个被遮蔽的子树重新绑回去、**静默暴露 deniedRoots**。原话点破："An ancestor carve-out ... re-binds the whole subtree on top of the mask and silently re-exposes the denied roots"。我们要在策略装配期就断言死这条（`check-sandbox.ts`）。
2. **哨兵环境变量** `LETTA_SANDBOX`：已在沙箱内的子进程靠它识别"自己被包过了"，`re-exec` 时**避免二次包裹**。对多 agent / 子 agent 嵌套的系统是必需的——我们 `task` 派生子 agent 时一定会踩。
3. **同一套机制，按数据敏感度给不同默认值**：memory 子代理默认沙箱化，agent 自己的 shell 默认不沙箱。我认为这是整个调研里最优雅的一条产品判断，直接抄（见 §7.2 的 profile 默认矩阵）。

### 2.6 OpenClaw：**默认进容器 + 可解释的显式提权**（终端产品最值得抄的模板）

- `ensureSandboxContainer` 语义 = "不存在就创建、存在就复用"（**会话级复用**，不是每次 tool call 新建）。
- 提权要连续过两道门：`requestElevated` 默认关，开启还需 `enabled` **且** `allowFrom.<provider>`；失败时**错误信息把每一道闸逐条列出来给你看**，并明确"哪一道没过"。
- 运行时标签本身就是立场声明：`const runtime = defaults?.sandbox ? "sandboxed" : "direct";`——**这个字符串应该出现在日志和 UI 里**。

### 2.7 dsh：**能力收窄是有意声明的**，另有三条凭据纪律

- 沙箱后端：`bwrap` / `Landlock` / `Seatbelt` / `Win ACL`；但**词汇表只管文件读写效果**，"网络访问与进程可见性"被**明确写在词汇表之外**（`sandbox/src/index.ts:27`）。声明边界比含糊覆盖诚实——网络不管确实是短板，但至少不会让人误以为安全。
- **升权走封闭词汇**：每个模式在**执行期**检查（**不烘焙进工具 schema**），非扩权请求**永不提示人类**，结果词汇是 `cancelled` / `unavailable` 这类封闭集，缺失与异常一律归一化为"不可用"（`sandbox/src/escalation.ts`）。审批事件成对追加，有提交与回放边界包夹，可审计。
  > 这与我在 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §1.3 抄的 Deep Agents"装配时隐藏"**看似冲突**，其实是两层不同的问题：装配期过滤决定**面上有什么**（防止模型去试做不到的事），执行期检查决定**这一次能不能升权**（防止策略漂移）。**我们两个都要**，顺序是：装配期按能力过滤 → 执行期按策略裁决 → 都不行才问人。
- 凭据三条（`credentials/src/index.ts`）：配置文件**只存环境变量名（引用），值由 provider 持有**；**每次操作重新解析引用，不跨操作缓存**（凭据轮换立即生效）；配置界面**只描述引用，绝不展示值**。
- 审批姿态：**`ask`，且 fail-closed——审批支持缺席时，`ask` 直接转 `deny`**。七家里唯一可以放心托管的缺省语义。

### 2.8 云端沙箱厂商：SDK 表面正在收敛成 E2B 形状

E2B（Firecracker microVM）的 **TypeScript SDK** 就是事实标准，连阿里云函数计算"Agent Sandbox"都是做 **E2B 兼容层**而不是自创 API：

```
Sandbox.create(template, timeout, metadata) / connect(id) / list() / getInfo()
  / setTimeout(ms) / pause() / kill() / uploadUrl(path) / downloadUrl(path) / getHost(port)
sandbox.commands.run(cmd) / list() / connect() / sendStdin() / kill()
sandbox.pty          ← 只有交互式 CLI、彩色输出、进度条、依赖 TTY 判断的命令才用它
sandbox.files.list/exists/getInfo/read/write/makeDir/remove/rename/watchDir
template 增删改查 / 从镜像构建 / tag-version-alias
```

选型不靠厂商宣传的启动时间，靠**突发并发下的 P95/P99 与成功率**（公开压测：Vercel 中位 0.67s / P99 1.12s / 成功 100%；Modal 中位 0.88s；E2B 中位 1.61s；Cloudflare 5s+；**Daytona 单实例顺序启动 0.1s 级最快，但 100 次突发并发成功率掉到 37%**）。
状态语义也不统一：Daytona 容器类 `stop` 保留文件系统但**清空内存**，VM 类保留内存可暂停恢复与分支；Modal 文件系统快照与内存快照分开（内存快照仍标 Alpha）。
> 结论：**远程沙箱 adapter 的默认清理策略必须显式声明"保留什么"**（fs / 内存 / 都没有），不能靠"stop 一下"这种含糊表达。

### 2.9 横向速查

| 项目 | 默认姿态 | 隔离层 | 网络 | 续跑 | 凭据 |
|---|---|---|---|---|---|
| Codex | 笼子优先（`workspace-write`） | 3+4 三平台原生 | **MITM 代理 + 域名白名单 + 凭据注入** | rollout / thread | 系统 keyring，进程加固保证不进子进程 |
| Claude Code | **默认关**（`enabled:false`），沙箱内自动放行 | 3+4（仅 Bash；全进程要另装 runtime） | 域名白名单 + proxy 端口，**首次新域名弹窗** | session resume | env deny |
| gemini / qwen | 关，`-s` 才开 | 4 或 5（六后端可选） | Seatbelt profile 三档；容器 `--network=none` | worktree checkpoint | key 不进沙箱 |
| Deep Agents | `StateBackend`（无 shell）；`LocalShellBackend` **裸跑** | 交给 provider | **不提供**（靠 backend 选环境） | LangGraph checkpoint | 明确"key 不进沙箱" |
| OpenClaw | **默认容器** | 5 | 容器边界 | 会话→容器 registry | 提权双门 |
| letta-code | **按数据敏感度分档**（memory 默认沙箱、shell 默认不） | 4（纯路径策略） | 不管 | cloud 沙箱 | 运行时注 env |
| dsh | **ask + fail-closed** | 3/4/ACL，**只声明文件效果** | 声明不管 | 事件回放 | 引用化、不缓存、不展示 |
| opencode | ask（但**明示不提供隔离**） | 无 | 无 | — | `.env` 读取需确认 |
| kimi-code | manual，**hooks 超时放行（fail-open）** | 无 OS 沙箱 | 回环 + bearer | — | **敏感文件硬过滤**（唯一做到代码里的） |

---

## 3. 抄哪些：九条机制（每条都有出处和落点）

1. **笼子优先，弹窗兜底**（Codex）。默认 `workspace-write`，而不是"每条命令问一次"。审批是稀缺资源，用多了就失效。→ 落 §7.2 profile 默认。
2. **能力探测决定可见面**（Deep Agents）。没有 shell 能力就别让 `bash` 出现在 schema 里。→ 与 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §1.3 同一条律，`sandbox` 是它的第三个来源。
3. **遮蔽用"缺席"而不是"拒绝"**（letta `--tmpfs`）。`denyRead: ['~/.ssh']` 应该让目录**不存在**，而不是"存在但报权限错"——后者等于给模型递了一张"这里有秘密"的地图。
4. **违规信息回传给模型**（Claude Code）。沙箱拦截时，把"哪个路径 / 哪个域名被拒"写进 tool result——这是模型能自我修正的前提，也是"错误是数据"的具体形态。
5. **域名策略进 system prompt**（Codex `<network>` 标签）。模型知道能连哪里，就不会白花 step 去试。**但注意**：这是 prompt 面变更，会击穿缓存前缀，所以要放进**稳定段**、且只在沙箱配置变化时变（对齐 [CONTEXT_BUDGET_MEMORY.md](./CONTEXT_BUDGET_MEMORY.md) §1.3）。
6. **`--die-with-parent` + 哨兵 env**（letta）。前者防僵尸沙箱，后者防嵌套重入。我们 `task` 派子 agent、MCP 起 stdio 子进程都会碰到。
7. **按数据敏感度给不同默认值**（letta）。同一个工具在不同调用方那里可以有不同的沙箱默认——memory 写盘默认关进笼子，用户显式请求的 shell 默认宽松。
8. **失败姿态必须显式选**（Codex fail-closed vs Claude Code 静默降级）。我选 **fail-closed 为默认**，理由：静默降级等于"你以为有闸门，其实没有"，而我们对外卖的是可信 harness。开发期想降级要显式写 `{ fallback: 'degrade-with-warning' }` 并**把降级事实落进 session log**。
9. **续跑是沙箱价值的一半**（几乎所有家都做了 `snapshot/hydrate`）。Agent 任务是长周期的，沙箱不光要关住 AI，还要保证"重启后还记得自己干到哪"。**忘了快照与恢复能力的沙箱只完成了一半工作**——这正好接到我们 v2.1 已经立了接口的 `ctx.checkpoints`（沙箱快照是它的第三个状态栈，见 §6.3）。

另外两条"便宜到不该省"的：
- **敏感文件硬过滤**（kimi `tools/policies/sensitive.ts`）：`.env` / `id_rsa` / `id_ed25519` / `credentials` / `.aws/credentials` 及 `.bak/.pem/.key/.old` 变体在**工具层**屏蔽读写，`.env.example` 与 `.pub` 豁免。它是防"提示注入外泄凭据"最便宜的一道闸，**和 OS 沙箱正交**——一期就要有。
- **破坏性判定用确定性规则前置**（qwen）：正则硬阻断放在**任何模型判断之前**。LLM 分类器误判不能放行破坏性命令。

---

## 4. 我们的方案：`ctx.sandbox` seam

### 4.1 一个要先定的架构裁决

Deep Agents 把 sandbox 做成"一种 Backend"（fs 与 execute 都在 sandbox 里）。**我们偏离它**，但保留它的实质：

> **`ctx.sandbox` 不是第三个工具，而是"执行世界的边界"。`ctx.fs` 与 `ctx.subprocess` 的默认 provider 由 sandbox **派生**——换一个 sandbox，文件工具和 `bash` 一起搬走。**

为什么必须这样：如果 sandbox 只管 `bash`，那 `read_file` 读的还是宿主磁盘，就会出现 **"模型 `bash` 在容器里 `cat foo.txt` 是空的，但 `read_file` 读到了内容"** 这种语义分裂——比没沙箱更危险，因为它会训练模型相信一个不存在的世界。这正是 [ARCHITECTURE.md](./ARCHITECTURE.md) 里"fs / subprocess / sandbox 共享同一个执行世界"那条的兑现处。

实现形状：sandbox provider 交出两个视图，`plugin-fs` / `plugin-subprocess` 把它们各自注册成 `ctx.fs` / `ctx.subprocess` 的默认 provider。**一个 seam 换，两个跟着换；工具名与描述不变**（对齐 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §2.3 "替换"那一行）。

### 4.2 `SandboxBackend` 的接口面（Definition）

```ts
// @agentic/sandbox
export type SandboxMode = 'off' | 'read-only' | 'workspace-write' | 'full-access';
export type NetworkMode = 'none' | 'proxy' | 'open';

export interface SandboxCapabilities {
  fs: boolean;            // 能不能提供文件视图
  shell: boolean;         // 能不能跑命令  → 决定 bash 在不在面上
  pty: boolean;           // 交互式（默认 false，只有远程/容器后端给）
  network: NetworkMode;   // 生效的网络档
  snapshot: boolean;      // 能不能续跑   → 决定 checkpoints 能否覆盖世界状态
  resourceLimits: boolean;// 能不能限 CPU/内存 → 决定预算层能否硬约束
}

export interface SandboxBackend {
  readonly id: string;                    // 'native' | 'container' | 'e2b' | 'none'
  readonly mode: SandboxMode;
  readonly capabilities: SandboxCapabilities;

  /** 探针：平台到底能不能用、缺哪个依赖。doctor 与装配期都调它 */
  probe(): Promise<SandboxReport>;

  /** 两个派生视图：plugin-fs / plugin-subprocess 注册为 ctx.fs / ctx.subprocess */
  openFs(scope: FsScope): FsBackend;
  openShell(scope: ShellScope): ShellBackend;

  /** 续跑锚点（B 类隔离由 ctx.checkpoints 负责，这里只管 A 类的世界状态） */
  snapshot(): Promise<SandboxSnapshot>;
  hydrate(at: SandboxSnapshot): Promise<void>;

  dispose(): void;                        // 由 ctx.effect 注册，保证无僵尸进程
}
```
`ExecResult` / `ReadResult` 等仍遵守 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §5.2 那条统一律：**错误是数据，不 throw**。沙箱违规也是一条 `ok:false`，带 `next: '被拒路径 ~/.ssh；如需访问请请求 read-outside 升权'`。

### 4.3 策略形状与覆盖算法（`defineSandboxPolicy`）

策略是**可序列化数据**（必须能进 `dumpConfig()`），不是 class 实例：

```ts
const policy = defineSandboxPolicy({
  mode: 'workspace-write',
  filesystem: {
    writable:   ['.'],                        // 工作区
    readonly:   ['./package.json', './src'],
    deny:       ['~/.ssh', '~/.aws', '.env'], // 遮蔽＝缺席（tmpfs / 不 bind），不是 EACCES
  },
  network: { default: 'deny', allow: ['*.npmjs.org', 'github.com'] },
  limits:  { cpu: 2, memoryMb: 2048, diskMb: 1024, timeoutMs: 120_000, maxOutputBytes: 100_000 },
  escalation: { ask: true, allowUnsandboxed: false },   // 见 §4.6
});
```

**覆盖算法定三条，写进单测**（把 letta 的"顺序即语义"和 Claude Code 的"窄者胜 + deny 在宽 allow 内仍生效"合并）：
1. **路径越窄越优先**；特异性相同则 **deny 胜**。
2. **一个宽 `allow` 永远不能把更窄的 `deny` 放回来**（`allow: ['~/'] + deny: ['~/.env']` → `.env` 仍不可读）。
3. **可写 carve-out 不得是任何 `deny` 根目录的祖先**——装配期直接报错，不等到运行期静默暴露（§3.1 的 bwrap last-mount-wins 坑）。

`read_file` 与 `bash` 里 `cat` 的可见性必须由**同一份 policy 渲染**，不能一个用 `ctx.fs` 过滤、另一个靠 OS 边界——否则工具层和内核层会给出两套答案，而**内核层是唯一有话语权的那一个**：工具层的 `deny` 列表应当被翻译成后端参数（SBPL 的 `(deny file-read* (subpath ...))` / bwrap 的 `--tmpfs` / 容器 mount），工具层只做"提前给出更好的错误信息"。

### 4.4 能力探测 → 工具面（与 BUILTIN_TOOLS 的接合点）

| 探到的能力 | 面 | 说明 |
|---|---|---|
| `capabilities.shell === false` | `bash` **不在 schema 里** | 不是"调了报错" |
| `capabilities.network === 'none'` | system prompt 的 `<network>` 段写"无外联" | 让模型别试 |
| `capabilities.snapshot === false` | `fork` / `rollback` 只回消息，并在结果里**明写世界状态未回滚** | 对齐记忆里那条坑 |
| `capabilities.pty` | 是否给 `pty` 参数 | 交互式命令、TUI 测试才用 |

### 4.5 `probe()`：先跑探针，再收紧规则

装配 agent 时（不是每次 tool call）跑一次：
`OS 与版本 → 后端可用性（seatbelt 的 `sandbox-exec` 存在？bwrap 版本？docker daemon 就绪？WSL1 还是 WSL2？）→ 内核特性（Landlock 是否 enable、user ns 是否允许）→ 路径是否存在 → 端口是否可绑`，产出一个 `SandboxReport`：

```
sandbox: native (bwrap 0.11.2)  mode=workspace-write  net=proxy
  fs: ok   shell: ok   snapshot: unavailable (no overlayfs)   limits: partial (cpu ok, memory needs cgroup v2)
  degraded: memory limit not enforced — cgroup v2 missing
  refused:  deny ~/.ssh → tmpfs mask applied
```
这个报告必须：① 出现在 `agentic doctor`；② 作为 durable 事件 `sandbox/probe` 进 session log；③ 任一项 `unavailable` 且 policy 要求 `failClosed` 时，装配直接失败并列出**没过的那道门**（抄 OpenClaw 的错误文案设计）。

### 4.6 升权（escalation）——我选择比 Claude Code 更硬的默认

Claude Code 的 `dangerouslyDisableSandbox` + `allowUnsandboxedCommands: true`（默认开）意味着：**沙箱只是一个建议**，模型可以自己申请出笼。我认为对"对外开放 SDK"这个定位太软。

我们的规则：
1. 默认 `allowUnsandboxed: false`（等价 Strict sandbox mode）。
2. 被拦时违规信息回传模型（§3.4），模型**可以请求升权**，但请求走的是 **`interrupt/approve` 那条 Submission 通道**（[LOOP_ENGINEERING.md](./LOOP_ENGINEERING.md) §5.4 枚举化），落 log、需人裁决，词汇封闭：`granted | denied | cancelled | unavailable`（dsh）。
3. **非扩权请求永不打扰人**（dsh）：模型换个合法路径重试、缩小 glob 范围这类，不该弹窗。
4. `excludedCommands`（"这个命令我知道要出笼"）**只能来自 profile / 用户配置，不能来自模型输出**——否则等于把逃生舱钥匙塞进笼子里。

---

## 5. 沙箱 SDK 的对外表面（你要的"能导入、方便执行沙箱"）

### 5.1 包与导入路径（遵守 create/define/with 动词铁律）

```
@agentic/sandbox                  # 纯契约 + 策略 + 探针类型 + 错误词汇（零厂商依赖）
  ├─ createSandbox / defineSandboxPolicy / withSandbox
  ├─ /native      # Seatbelt / bwrap / Win 受限令牌（我们自己的实现）
  ├─ /container   # docker / podman / devcontainer
  ├─ /remote      # e2b / daytona / modal adapter（E2B 兼容为互操作轴）
  ├─ /iso         # B 类：CoW 工作区视图（供 checkpoints / teams 复用）
  └─ /testing     # FakeSandbox（CI 用，不依赖真 OS）

@agentic/plugin-sandbox            # ctx.sandbox 的 seam 与装配，派生 ctx.fs / ctx.subprocess
```

**两种用法都成立**——这是"SDK 而不是内部模块"的关键：

```ts
// 用法 1：给 agent 用（主路径）——一行声明笼子，工具面自动跟着变
import { createReactAgent } from '@agentic/core';
import { nativeSandbox } from '@agentic/sandbox/native';

const agent = createReactAgent({
  profile: 'core8',
  sandbox: nativeSandbox({
    mode: 'workspace-write',
    deny: ['~/.ssh', '.env'],
    network: { default: 'deny', allow: ['*.npmjs.org'] },
    failClosed: true,
  }),
});

// 用法 2：自己拿沙箱当执行环境用（不接 agent）——create* 就要 dispose
import { createSandbox } from '@agentic/sandbox';
import { containerSandbox } from '@agentic/sandbox/container';

await using sbx = await createSandbox(containerSandbox({
  image: 'node:22-slim', network: 'none', limits: { cpu: 2, memoryMb: 2048 },
}));
const r = await sbx.exec(['npm', 'test']);     // 结构化结果，不 throw
```

字符串简写 `sandbox: 'native:workspace-write'` 也接受（内置 profile 就这么写），但 `dumpConfig()` 必须打印它展开后的完整 policy 对象——**简写是便利，不是不可见**。

### 5.2 与既有 seam / 通道的关系（别处不用改，这里点名）

| 已有设计 | 沙箱带来的变更 |
|---|---|
| `ctx.fs` 七方法 | 由 sandbox **派生**（`openFs`）；`virtual_mode` 那层路径约束**保留**，但它只是"体验优化"，**安全边界在 sandbox** |
| `bash` 工具 | `requires: ['shell']` → `capabilities.shell` 由 sandbox 报；**`scout` profile = `read-only` 笼子，`bash` 依然存在但只允许白名单只读命令**（比"没有 bash"更有用） |
| `ctx.checkpoints` | 世界状态的第三层：影子 git（文件内容）+ **sandbox snapshot（进程/内存/已装依赖）**；`capabilities.snapshot===false` 时必须明写未回滚 |
| `teams.worktree` | 属 **B 类**，二期可由 `/iso` 的 CoW 后端加速（APFS clonefile / overlayfs / btrfs / reflink / ProjFS / git worktree 兜底），语义不变 |
| 通道 C `run_script`（Code-as-action） | **原为二期，现在提前有条件地可行**：只要 sandbox 就位，`run_script` 就是"沙箱里写个脚本跑一遍"。裁决：一期只给 `native`/`container` 后端开，`off` 模式下该工具不在面上 |
| `policy`（ask/deny） | policy = **意图层裁决**（要不要问人），sandbox = **能力层边界**（问了也做不到）。顺序：能力过滤 → 意图裁决 → 内核边界 → 网络代理 |

### 5.3 进程与资源（Bun 特有的一段）

- ★ **argv 优先，不拼 shell（铁律）**：**harness 自己发起的**执行（`git` / `rg` / `docker` / `sbx.exec([...])`）一律 `Bun.spawn(argv: string[])`，**绝不自己拼一个 shell 字符串**。只有模型经 `bash` 工具主动提交的命令才需要 `sh -c`——而它必然在笼子里跑，这正是沙箱存在的前提。Deep Agents 的 `LocalShellBackend` 正是栽在 `shell=True` 上（§2.4）：它把两者混成了同一个通道。
- 用 `Bun.spawn`；`--die-with-parent` 在 macOS/Linux 交给 bwrap/seatbelt 自身，**Windows 无对应**，必须显式做进程树回收（`taskkill /T /F`），与我们 MCP stdio 传输那套同一份实现（复用，不写两遍）。
- 超时与输出上限是**沙箱层的默认值**（抄 Deep Agents：`timeoutMs=120_000`、`maxOutputBytes=100_000`），超了截断并在结果里写"已截断，可用 grep/收窄路径"。
- 资源限制（CPU/内存/磁盘）在 native 档**做不到**（Seatbelt/bwrap 不管 cgroup）。**这一档必须如实报 `limits: false`**，别让预算层以为有硬约束——否则一个失控的 `npm install` 就能吃满宿主。要硬约束就选 `container` / `remote`。

---

## 6. 纵深：四道闸各自挡什么（把 v2.1 的"三道闸"升级）

```
1  能力过滤（装配期）   做不到的事不在模型可见面上          ← Deep Agents / sandbox probe
2  意图裁决（policy）   要不要问人：allow / ask / deny      ← 我们 ctx.policy + kimi 敏感文件硬过滤
3  OS 边界（sandbox）   问了也做不到：路径缺席 / syscall 拒绝 / 资源上限   ← Seatbelt / bwrap / 容器 / microVM
4  网络出口（proxy）    能连哪儿 + 密钥不进笼子              ← 域名白名单，二期 MITM + 凭据注入
```

一条不变量贯穿：**四道闸的生效结果都要进 session log**（"模型可见即已记录"的延伸——**被拦住的东西同样要可回放**，否则无法评测、无法复盘"为什么这次没成"）。

### 6.3 与"回退/分支"的接线（这里补上 v2.1 缺口）

`sandbox` 决定 `ctx.checkpoints.snapshot(scope)` 能覆盖到哪一层：

| scope | 覆盖 | 依赖 |
|---|---|---|
| `messages` | 消息投影 | 总是可用 |
| `workspace` | 文件内容 | 影子 git（一期） |
| `world` | 文件 + **已装依赖 + 进程状态 + 网络配置** | `capabilities.snapshot`（容器/microVM 才有） |

**没有 `world` 时，`fork` 出两个分支跑 `npm install` 结果不可比**——这条要在 `rollback()` 的返回值里显式说出来，而不是静默假装成功。

---

## 7. 一期 / 二期切片，以及在**你这台 Windows 机器**上的现实

### 7.1 切片

**一期（必做，且是 `bash` 上面的前提）**
1. `@agentic/sandbox` 契约 + `defineSandboxPolicy` + 覆盖算法三条 + `check-sandbox.ts` 断言集。
2. `/native`：**macOS Seatbelt**（SBPL 模板 + 参数化路径）与 **Linux bwrap**（`buildBwrapArgs` 顺序照 letta，含 `--tmpfs` 遮蔽与 `--die-with-parent`）；`/proc` `--dev` 重置防泄漏宿主进程状态。
3. `probe()` + `SandboxReport` + `agentic doctor` 显示 + `sandbox/probe` durable 事件；**fail-closed 默认**。
4. sandbox 派生 `ctx.fs` / `ctx.subprocess`（一个换、两个跟着换）。
5. 哨兵 env `AGENTIC_SANDBOX=1` 防重入；Windows 进程树回收。
6. 网络 `none` / `proxy`（域名白名单 + 本地 HTTP CONNECT 代理，**不做 MITM**）；`<network>` 段进 prompt 稳定区。
7. **kimi 式敏感文件硬过滤**（`.env` / `id_*` / `credentials` / `.pem/.key/.old` 变体）——便宜、立即、和 OS 沙箱正交。
8. `.agentic/sandbox.json` + `templates/dot-agentic/` 里给一份可直接用的默认策略。

**一期建议同时做（server/无人值守形态需要）**
9. `/container`：docker/podman，默认 `--network none --memory --cpus --cap-drop ALL`；`ensureSandboxContainer` 式会话复用；`.agentic/sandbox.Dockerfile`。

**二期**
10. `/remote`（E2B 兼容为第一适配，Daytona/Modal 跟进）、`world` 级 snapshot/hydrate。
11. `/iso` CoW 工作区视图（APFS clonefile / overlayfs / btrfs / reflink / ProjFS / worktree 兜底）→ 加速 `checkpoints` 与 teams 并行。
12. Windows **受限令牌**原生后端；MITM 凭据代理（密钥不进沙箱，转发时注入）；Codex 式 `execpolicy`（WASM/OPA 命令规则引擎）；seccomp 精细规则集。

### 7.3 现实结论（先看这条再决定一期做什么）

你的开发机是 **Windows**，这会直接决定"一期做完能不能自己跑起来验证"：

| 环境 | native 档可用性 | 说明 |
|---|---|---|
| macOS 12+ | ✅ 开箱 | 系统自带 `sandbox-exec`，零依赖 |
| Linux（内核 ≥ 5.13，user ns 开） | ✅ 装 `bubblewrap` | 我们的主力 CI 环境 |
| **Windows 原生** | ❌ 一期**没有** | 受限令牌要 Win32 能力，我排在二期；一期这里会 `unavailable` |
| **WSL2** | ✅（走 Linux 分支） | 明确拒绝 WSL1（bwrap 与 WSL1 内核兼容层不兼容，Claude Code 同样拒绝） |
| Docker Desktop | ✅ | 容器路线，Windows 上唯一现实的强隔离 |

**所以一期你在本机验证沙箱，只有两条路：WSL2 里跑 bwrap，或 Docker Desktop 跑 `/container`。** 这条必须写进文档和 `SECURITY.md`，并且 `probe()` 要能给出"这台机器上你该用哪个后端"的建议——**含糊覆盖比明确声明不提供更有害**（opencode 的教训）。

---

## 8. 需要你拍板的 4 点

1. **默认姿态**：一期默认是否直接 `native / workspace-write / fail-closed`？（我建议：`sdk` 与 `core8` profile 默认 `workspace-write`；`scout` 默认 `read-only`；**任何 `off` 必须显式声明并在 TUI 顶部常亮警告**。备选：一期默认 `off` + 首次运行强提示。）
2. **升权逃生舱**：`allowUnsandboxed` 默认 **false**（我建议）还是学 Claude Code 默认 true？默认 true 体验顺很多，但等于"沙箱是建议"。
3. **容器后端是否进一期**：进（我在 Windows 上就能立刻验证，且无人值守形态需要）还是纯二期（一期只做 native，Windows 用户先看到 `unavailable`）？**我倾向进一期，但只做 `docker`/`podman` 两个 provider，不做 devcontainer 定制。**
4. **`/iso` CoW 与 `checkpoints` 影子 git 的关系**：一期就抽象成同一个 `WorkspaceView` interface（影子 git 是它的兜底实现），还是先各自实现、二期再合流？**我建议一期就抽象接口**，否则 `teams` 的并行隔离与 `checkpoints` 的回滚会写成两套。

## 9. 我们**不做**的（拒绝清单）

- **不做解释器级软沙箱**（AST/import 白名单）当成安全边界——连 smolagents 作者都写进 docstring 承认它不是。我们要 code-as-action 就直接走真笼子。
- **不把 sandbox 实现成第三种工具面**（`sandbox_run`/`sandbox_fs` 一堆新工具）——工具面 7 个原子工具不变，变的只是它们背后在哪个世界执行。
- **不在一期做 MITM 凭据代理**（Codex 那条最漂亮但也最重，需要证书链、域名规则、审计）。一期只做"密钥不进沙箱 + 白名单放行"。
- **不支持 WSL1**，不做 Windows 原生受限令牌（二期），并在 `doctor` 里明说而不是降级到"看起来能跑"。

## 10. 资料来源（本次核实）

Codex `codex-rs/sandboxing`（`manager.rs`、`seatbelt.rs` + `.sbpl`、`linux-sandbox/`、`landlock.rs` legacy、`windows-sandbox-rs`、`network-proxy/`）；Claude Code 官方沙箱文档与 `@anthropic-ai/sandbox-runtime`；gemini-cli / Qoder CLI / qwen-code 的六后端与 `sandboxConfig.ts`、Seatbelt profile 矩阵；Deep Agents `backends` 文档（`SandboxBackendProtocol`、`LocalShellBackend` 风险清单、`dcode --sandbox`）；letta-code `FsSandboxPolicy` 与 `buildBwrapArgs`；OpenClaw `ensureSandboxContainer` 与提权双门；dsh `sandbox/src/{index,escalation}.ts` 与 `credentials/src/index.ts`；kimi `tools/policies/sensitive.ts`；opencode `SECURITY.md`；"7 大开源 Agent 源码对比·安全与权限控制"、"智能体沙箱是怎么做的：跨框架源码拆解"、"AI Coding Agent 沙箱的设计原理解析"；E2B / Daytona / Modal / Vercel / Cloudflare 公开 SDK 与 2026-08 突发并发压测；阿里云 Agent Sandbox 的 E2B 兼容说明。

> ⚠️ 同 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §0 的版本漂移教训：沙箱这块的**默认值**是产品级决策、改动很频繁（Claude Code 的 `failIfUnavailable`、`allowUnsandboxedCommands` 都是近期才加的）。**任何"某家默认开/关沙箱"的说法都必须回源码或当期官方文档核对**，我们的能力面受 `check-sandbox.ts` 断言而不是受文档一句话约束。
