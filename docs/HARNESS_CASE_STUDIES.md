# Harness 案例研究：dsh / pi / Codex / 分支回退系

> 状态：**规划稿 v1**
> 目的：看四个成熟项目如何把「不确定的模型」翻译成「可落地的工程制品」。我们只借鉴思路，不复刻实现。
> 阅读前提：先看过 [ARCHITECTURE.md](./ARCHITECTURE.md) 的第 2、3 节（内核五原语与 seam 清单），否则本文的"落点"无法对齐。

---

## 0. 为什么挑这四个

每个 harness 本质上都在回答同一道题：**模型不可靠，你的系统凭什么可靠。**

但这四家给出的答案差别极大——这恰恰说明「怎么可控」是**设计选择**，不是物理定律。把它们并排看，比单独精读任何一家都有价值，因为差异处就是取舍处，取舍处就是我们的作业。

| 项目 | 一句话定位 | 它把"重量"压在哪里 |
| --- | --- | --- |
| **dsh**（DeepSeek Harness） | 一切皆插件的 Agent 运行时 | 压在**可撤销**与**可追溯**上 |
| **pi**（badlogic/pi-mono，现 earendil-works/pi） | 极简可组装的 harness 乐高 | 压在**缩小能力面**与**校验回填**上 |
| **Codex CLI**（openai/codex，Rust） | 单内核多前端的生产级 coding agent | 压在**边界纵深**与**协议契约**上 |
| **分支回退系**（pi 会话树 / Codex ThreadRollback / Shadow Git / worktree 隔离） | 让 agent 可以悔棋的一族机制 | 压在**时间轴**上 |

一个观察收束全篇：**这四层重量互不冲突，可以叠加。** 缩面（pi）让行为可预测，纵深（Codex）让行为不越界，真相源（dsh）让行为可解释，时间轴（回退系）让错误可撤销。我们四个都要，这就是本文的输出。

---

## 1. 横向对照表

| 维度 | dsh | pi | Codex | 回退系 |
| --- | --- | --- | --- | --- |
| 核心隐喻 | 插件树 + `ctx` | 乐高库 + 学徒 | OS 内核 + 系统调用 | 时间机器 |
| 内核是否有特权 | **无**（模型适配器、工具注册表、session log、agent loop 全是插件） | 无特权内核，但**分层单向依赖**（ai → agent → coding-agent → tui/web-ui） | **有**（`codex-core`），靠治理约束膨胀 | — |
| 防止内核膨胀的手段 | 架构约束（kernel 不依赖 plugin） | 每层可单独替换/单独使用 | `AGENTS.md` 明文 **"resist adding code to codex-core"** + 新功能优先放能力 crate | — |
| 模型面（prompt/工具） | systemPrompt 分层装配插件 + 渐进披露 | **< 1000 token** + 4 个工具（read/write/edit/bash） | 数千 token + `exec_command`/`apply_patch`/`update_plan`，**与模型协同设计** | — |
| 状态真相源 | append-only JSONL session log + projection seam | JSONL，每行一个事件，**带 `id`/`parentId` 成树** | `rollout` JSONL + 反向扫描器（从文件尾倒读定位状态） | checkpoint 序列 |
| 悔棋能力 | `fork` | `/tree` `/fork` `/clone` | `Op::ThreadRollback`、`--branch` | 对话树 + **影子 git** |
| 安全边界 | `ctx.policy` + `tools/pre-execute` waterfall | **几乎无**（进程内工具权限等同 shell） | **四道闸**：approval → execpolicy(WASM/OPA) → OS 沙箱(Seatbelt/Landlock+seccomp/Win restricted token) → MITM 网络代理 | — |
| 人对 agent 的干预 | waterfall 事件可短路 | 斜杠命令 + 扩展代码 | **`Op` 枚举（约 20 变体：Interrupt/TurnInput/RecoverTurn/Compact/ExecApproval/PatchApproval/ThreadRollback/Shutdown）** | — |
| 扩展形态 | plugin + bundle + patch | 进程内 `.pi/tools/*.ts` + `.pi/commands/*.md` + MCP | MCP + skills + config | — |
| 多模型策略 | provider 插件 | **库而非代理**：超集类型，provider 特性不做有损转换 | Responses API（WS 优先，失败回退 HTTP SSE） | — |
| 可观测 | typed events + telemetry | **grepability**：`grep` + `jq` 直接看内部状态 | OpenTelemetry + insta 快照测试 | — |
| 适合谁 | 做生态/做产品 | 做**自己的** agent / 内部工具 | 生产 coding agent | 探索式长任务 |

> 表中「dsh 模型可见即已被记录（Model-visible means logged）」、「pi 短 prompt 的三个赌注」、「Codex 四道闸」是各家最不可妥协的一条，本文第 2 节逐个展开。

---

## 2. 逐个拆：它们各把哪一块不确定性变成了什么制品

### 2.1 dsh —— 不确定性 → 「一切变更可撤销」+「唯一真相源」

四条转化：

| 不确定性 | 变成的工程制品 | 为什么这样就可控了 |
| --- | --- | --- |
| 改了行为，不知道会不会炸别处 | 注册即 `ctx.effect`，卸载按**逆序完整回滚** | "可撤销"才敢热重载、才敢在测试里隔离装配 |
| 它为什么这么说 | session log 的不变量：**模型看到的每个字都被记录**，历史是 log 的**纯投影** | 解释 = 回放，不是猜 |
| 策略喊了没用（写在 prompt 里的"请不要删文件"） | `waterfall` 事件监听者可**短路** | 拦截点从"请求模型"挪到"替模型决定" |
| 换沙箱要改一片代码 | capability seam 三角色（定义/提供/消费）+ fs/subprocess/sandbox 共享同一"执行世界" | 一处 provider 覆写，全局生效 |

**本次新增借鉴**：dsh 还有一组我们没覆盖的机制——**inbox（收件箱）**。它回答的是一个非常具体的不确定性：*"agent 正跑在第 30 步，用户中途插了一句『等等，别用那个库』，这句话算新任务、还是补充说明、还是打断？"*（社区解读；官方 architecture.md 未展开。）传统做法是塞进消息数组继续跑，于是要么插话被忽略，要么上下文被搅乱。dsh 的答案是把"输入到达"这件事本身事件化、排队、显式分类。

→ 落到本项目：补 `ctx.inbox` 与 `turn/input-queued` 事件。见 §5-1。

### 2.2 pi —— 不确定性 → 「把面缩到最小」+「校验错误回填而不是抛崩」

pi 的价值不在极简本身，而在于它给出了**极简的理由**，以及它**承认的代价**。

**四个真正可抄的点：**

1. **Terminus 观察**（Mario 自述的立项直觉）：Terminal Bench 上有个 harness 只给模型**一个工具**——跟 tmux 会话交互，模型得自己发按键、读 ANSI 序列。就一个工具，它常年排前三、经常第一。结论：**今天的模型已经被 RL 训练得天然知道"编码 harness 是什么"，不需要在上面堆指令。** 长 prompt 里大部分规则是在对抗两年前的模型缺陷，而那些缺陷已经不存在了。
   → 这条对我们的直接意义：**工具面数量是设计期的决定，不是接进来以后再说。**
2. **一份定义产出三样东西**（TypeBox 路线）：TS 类型（编译期）+ JSON Schema（发给模型）+ 运行时校验器（防幻觉参数）。**校验失败时把错误作为工具结果返回给模型，让它自己修正。**
   → 这条是 pi 最被低估的工程点。大量手搓 agent 的死法是"一次参数幻觉整个循环崩掉"。用一行 `isError: true` 的回填就能堵住。**必须写进我们的硬约束**（§5-5）。
3. **学徒 vs 万事通**：出厂几乎是"素"的，能力通过 `AGENTS.md` / `.pi/tools/` / `.pi/commands/*.md` 逐步教会，**并且这些东西以文件形式存在于你的仓库里，可以 git commit、可以 code review、可以在团队共享**。万事通模式下能力属于厂商；学徒模式下能力属于你，而且可版本化。
   → 这条与本项目定位（SDK）完全同向：我们的 `systemPrompt` section / `commands` / skills 都必须是**可落盘、可 diff 的文件制品**（§5-12）。
4. **库而不是代理**：网关类方案把各家 API 抹平成 OpenAI 格式，代价是有损（丢 Anthropic cache control、丢 Google context caching、丢 SSE 事件粒度）。pi-ai 用**超集类型**保留每个 provider 的独有特性，还用条件类型让"模型字符串决定返回类型"——不支持 reasoning 的模型上访问 `r.reasoning` **编译期报错**。
   → 这条解释了为什么"Agent 的 LLM 调用层必须是库"：agent 需要在模型**开始**调用工具的瞬间就开始渲染 UI（参数还没生成完）、需要精确 cache 命中率来排布上下文、需要流中断时精确恢复。这些粒度一旦被代理层翻译过就补不回来。
5. **grepability**（工程品味的判据，原文值得背下来）：一个系统的内部状态能不能用最普通的 Unix 工具（`grep`/`jq`）检查，是衡量它工程质量的重要指标。对比"加密 SQLite + 自定义二进制格式"的竞品，出问题时的可调试性天差地别。
   → 落 §5-4：把这条做成**守门脚本**，而不是审美偏好。

**pi 自己承认的三个坑**（这决定了我们不能全盘照抄）：
- 极简 prompt **依赖强模型**：接 30B 以下会明显"不主动"（不先读就改、不跑测试验证）。解法是在 `AGENTS.md` 里把工作流写细——等于自己干厂商的活，但至少**有地方干**。
- 进程内 `.pi/tools/*.ts` 权限等同 shell：**克隆了别人的仓库直接跑 = 执行仓库里的任意代码**，且加载路径隐蔽。
- JSONL 会话文件会膨胀（长会话 + 大文件读取轻松上百 MB），上下文压缩不会自动清理磁盘原始事件流。

### 2.3 Codex —— 不确定性 → 「干预动作枚举化」+「边界纵深」

1. **Op 进 / Event 出，内核不暴露函数式 API。** 前端与内核的交互**只发生在两个地方**：提交一个 `Op`，消费一条 `Event` 流。约 20 个 Op 变体里包含 `Interrupt`、`TurnInput`、`RecoverTurn`、`Compact`、`ExecApproval`、`PatchApproval`、`ThreadRollback`、`Shutdown`。
   → 这是"可控"最硬的工程表达：把**人对 agent 的一切干预收敛成一个可枚举、可序列化、可测试的封闭集合**，而不是散落在各处的回调和 `setXxx()`。散落的 API 你无法审计"人能对这个 agent 做哪些事"；枚举可以。落 §5-6。
2. **四道闸纵深**：`AskForApproval`(untrusted/on-request/granular/never) → 策略引擎 `execpolicy`（WASM/OPA 规则）→ OS 级沙箱（macOS Seatbelt / Linux Landlock+seccomp+bubblewrap / Windows restricted token）→ 网络 MITM 代理按请求授权。**任何一道失效，下一道仍在兜底。**
   → 这条打脸了"我们有一个 policy seam 就够了"的想法。policy 是**判断**，沙箱是**能力剥夺**，二者性质不同：判断可能被绕过，剥夺不会。落 §5-7。
3. **协议即契约 + `#[non_exhaustive]`**：`protocol` crate 独立存在，前端只依赖协议就能编解码全部消息，不必链接庞大的内核实现；协议枚举标记为非穷尽，允许**向后兼容地扩展**。
   → SDK 的接口稳定性纪律：我们的 exports 类型面应该同样保证"加变体不破坏下游"（联合类型 + `string & {}` 兜底 / 或显式的 unknown 分支要求）。
4. **`rollout` JSONL + 反向扫描器**：resume 的实现不是数据库，而是从文件尾部倒着读快速定位状态；`codex resume` 重建上午的全部上下文。JSONL 让 `jq` 能人工检视——**事件溯源带来天然可审计性**。
5. **传输也要有降级路径**：优先 WebSocket Responses API，失败自动回退 HTTP SSE（`FallbackToHttp`）。
   → 我们的 retry/backoff 分类（LOOP_ENGINEERING §4）只处理了"状态码语义"，没处理"通道降级"。落 §5-8。
6. **治理而不是架构**："resist adding code to codex-core" 是写进仓库 `AGENTS.md` 的明文规则。**微内核能不能守住，靠的是一条明文规矩 + CI，不是自觉。**
   → 光有 `kernel 不依赖 plugin` 的架构约束不够，要落到 AGENTS.md + `check-deps.ts`。落 §5-10。
7. **为什么不做 RAG**（原话很有价值）：coding agent 的"知识"主要是当前 repo 的文件系统本身，按需 `read_file` 即可，不需要预建向量索引。想加跨 repo 检索，**正确的扩展点是把检索器封装成 MCP server，而不是改内核**。
   → 与我们的 [REPO_INTELLIGENCE.md](./REPO_INTELLIGENCE.md) 有张力，裁决见 §4-3。
8. 额外一条：Codex 的 prompt 与模型（GPT-5 Codex 系列）是**配套演进、专为模型行为特征调优**的（`exec_command`、`apply_patch` 这两个工具名本身就是调优产物）。
   → 这解释了为什么我们不能白拿别人的 system prompt：prompt 与模型是成对的。我们的 `systemPrompt` section 必须能按 provider/model 分档。

### 2.4 分支回退系 —— 不确定性 → 「可回退的时间轴」，和那个必须抄的教训

**共同做法**：会话不是数组而是树。pi 每条 entry 有 `id`/`parentId`，`/tree` 在树里导航并**从任意历史节点继续**，`/fork` 从某条用户消息分叉出新会话文件，`/clone` 复制当前活动分支；两条探索路径留在同一个文件里可来回切换。Codex 有 `Op::ThreadRollback` 与 `--branch`（在指定 checkpoint 上开新分支继续）。这解决的正是"第 40 步犯了致命错误、41–50 步越修越糟"——人类此时会叹口气敲 `git reset --hard`，agent 必须也有这个能力。

**必须抄的那条教训**（社区踩出来的）：

> ⚠️ **树形回退只回退对话上下文，不回退文件系统。** `/tree` 管的是消息树，磁盘上被它改过的代码依然在。

于是"方案 A / 方案 B 平行探索"其实是个幻觉——**只有一个真实世界**，两个脑内剧本。你在节点 30 悔棋去试方案 B，工作树里还残留着方案 A 写下的 200 行改动，模型看不到这些改动的存在（它只看得到被回退后的消息投影），于是行为完全不可比。

社区的解法是 **Shadow Git**：在每个工具执行边界打快照，用 git 原生 branch + tag 承载分支语义（零额外存储成本），切分支时 `git checkout` 原子切工作目录，每个分支有独立 ledger，并需要 prune 防对象膨胀。TiDB 那场分享是同一个意思的极端版本：把 agent 的 workspace 当成**可版本化、可分支、可回滚、可审计、可授权、可跨沙箱迁移**的事务对象来管，用 MVCC 支撑多 agent 并行探索。

→ 结论：**回滚必须是跨两个状态栈的原子操作**——(1) 消息流投影、(2) 世界状态（文件系统 / git / 外部副作用）。少任何一个，fork 出来的分支都不可比，多 agent 并行探索会互相污染。这是本项目要**新增的一个 seam**：`ctx.checkpoints`。

配套一条：多 agent 并行探索的正确隔离单位是 **git worktree**（不是共享 cwd 的目录约定）。learn-claude-code 的教学终点 `s12_worktree_task_isolation` 就是它；dsh 的 subagent/jobs 也是同一方向。落 §5-11。

---

## 3. 共性提炼：六条转化律 + 一条元规律

我把四家重复出现的手法归纳成可检验的形式。每条都配一个"验收问题"——**答不出来的模块就是还没工程化。**

| # | 转化律 | 从不确定到确定 | 本项目的验收问题 |
| --- | --- | --- | --- |
| L1 | **收缩能力面** | "模型可能乱来"→"它可选项少" | 默认 profile 暴露了几个工具？能不能说清每个为什么必须在？ |
| L2 | **校验一切，错误回填** | "参数会幻觉"→"幻觉变成模型可读的反馈" | 非法参数走 throw 还是走 `tool_result(isError)`？有测试吗？ |
| L3 | **干预动作枚举化** | "人能乱动"→"人能做的动作是封闭集合" | 有没有一处能列出"人对 agent 的全部操作"？能不能序列化进 log？ |
| L4 | **唯一 append-only 真相源 + 纯投影** | "为什么这么说"→"回放" | 送给模型的消息数组，能否 100% 从 log 重新导出、无任何额外内存状态？ |
| L5 | **状态明文化、可 grep** | "出问题查不了"→"jq 能查" | 新人能不能用 `grep` 在会话文件里找出这次为什么花了 4 块钱？ |
| L6 | **变更可撤销，且跨全部状态栈** | "错了收不回"→"逆序回滚 / checkpoint 回退" | 回退到 step 30 时，磁盘状态和消息状态是不是**同时**回到 30？ |

**元规律：边界靠纵深，不靠单点判断。**（Codex 四道闸给的全部信息）任何"我们加了 policy 检查"的表述都值得追问一句：判断被绕过之后，还有什么兜底？

这七条会作为 `HARNESS_CASE_STUDIES` → `ARCHITECTURE` 的回填标准：**每个 seam 至少能对应到一条律。**

---

## 4. 它们互相打脸的地方，以及本项目的裁决

这一节是本文的主要产出。差异处必须选边，不选边就会做出一个四不像。

| # | 冲突 | 两边立场 | **我们的裁决** | 理由 |
| --- | --- | --- | --- | --- |
| 1 | 工具面该多大 | pi：4 个工具就够，多了破坏行为；dsh：多工具 + 渐进披露（`search_tools`/`call_tool`） | **分层**：`core` profile ≤ 8 个工具（不做渐进披露）；`coding`/`research` profile 才上多工具 + 披露；repo-intel 那 10 个工具属于 `coding` | SDK 的默认面就是用户的先验；把 4 个工具的极端情形做成可验证的默认档，重能力靠 profile 叠加 |
| 2 | system prompt 该多长 | pi：< 1000 token；Codex：数千 token 且与模型配套调优 | **给 prompt 装配加 token 预算并做 CI 断言**：`core` ≤ 1.2k，超出即失败；每次装配报告实际 token 数 | 这是 pi 赌注二的直接工程化——每个 prompt token 每轮重复计费并挤占真实上下文。断言比规范文档有效 |
| 3 | 要不要建代码索引 | Codex：**明确不做 RAG**，按需 `read_file` 即可；我们：[REPO_INTELLIGENCE.md](./REPO_INTELLIGENCE.md) 建四层索引 | **不冲突，但要收窄**：一期只做**结构图**（tree-sitter + 符号/调用图 + FTS5 驼峰分词），它不是向量 RAG，是图遍历；**跨 repo 语义检索走 MCP**（与 Codex 建议一致）；默认 profile **不建向量索引** | Codex 反对的是"为 coding agent 预建 embedding 库"，我们的核心工具是 `trace_path`/`impact`，本来就是图操作 |
| 4 | 进程内工具 vs MCP | pi：同机场景进程内 TS 文件比 MCP 简单一个数量级，"没有 server、没有独立进程、没有 JSON-RPC 握手"；我们：MCP 是重投入方向 | **两条都要，并给出显式判据**（写进 SDK_SURFACE）：同进程同机 + 可信代码 → `defineTool` 一个文件；跨语言 / 第三方 / 需要进程隔离 / 需要复用给别的 agent → MCP | 判据缺失时团队会两边都写，最后 MCP 化一切，把简单事情复杂化 |
| 5 | 有没有特权内核 | dsh：无特权，agent loop 也是插件；Codex：有 `codex-core`，靠 AGENTS.md 治理膨胀 | **走 dsh（无特权内核），但补 Codex 的治理**：架构上 kernel 不依赖 plugin + `scripts/check-deps.ts`，文化上 `AGENTS.md` 明文"resist adding code to kernel" | 我们是要开放生态的 SDK，"官方不接受外部 PR"的 dsh 都坚持 loop 可替换，我们没理由做特权内核；但纯架构约束历史上都守不住 |
| 6 | 安全强度 | pi：几乎无边界（承认是坑）；Codex：四道闸 + 三平台 OS 沙箱 | **一期做三道**：policy waterfall（判断）→ fs/world 的 cwd/路径约束（能力范围）→ subprocess argv 约束 + 网络开关（能力剥夺）；**OS 级沙箱与 MITM 代理列二期**，且 passthrough 必须在文档与 `doctor` 输出里标红"默认不安全" | SDK 不能假装安全，但也不能因为做不出 Seatbelt 就 0 道闸。诚实标注比默认 allow-all 更负责 |
| 7 | 语言与进程模型 | Codex：Rust 内核 + 多前端二进制；我们：TS/Bun 库嵌在用户进程里 | **保留"两通道"抽象，即使同进程也照样切**：所有外部交互走 `Op`（我们的 `agent.interrupt()/compact()/rollback()/approve()`）与 `Event`，不暴露散装 setter | 这样二期把 kernel 搬进 worker 进程或服务端时**不改对外 API**，等于提前把跨进程重构的成本付掉了 |
| 8 | 会话文件格式 | dsh/pi/Codex 都选 JSONL 明文；我们原方案也是 JSONL | **确认，并加一条硬约束**：任何"模型可见"内容不得只存在于压缩/加密/二进制表示里；artifact 卸载也必须是明文文件 + ref | 三家同时选 JSONL 不是巧合，是 L4/L5 的必然结果 |

---

## 5. 本次落到项目里的改动（12 项，按优先级）

| # | 改动 | 出处 | 落点 |
| --- | --- | --- | --- |
| 1 | 新 seam `ctx.checkpoints`：**跨状态栈的原子回滚**（消息投影 + 影子 git 快照），支持 `fork`/`rollback(to)`/`compare(a,b)` | pi 树形回退不回退文件的教训 + Shadow Git + Codex `ThreadRollback`/`--branch` | `packages/plugins/checkpoint/`、ARCHITECTURE §3 新增一行 |
| 2 | session log 从线性改**树**：事件带 `parentId`，`fork`/`revise` 从同一份 log 派生 | pi 会话树（`id`/`parentId`） | `plugin-session`，ARCHITECTURE §5 |
| 3 | 新 seam `ctx.inbox` + 事件 `turn/input-queued`：插话显式分类为 `append`\|`newTask`\|`interrupt` | dsh 的 inbox 机制 | ARCHITECTURE §3/§4 |
| 4 | 守门脚本 `check-grepability.ts`：会话文件必须可被 grep/jq 直接读懂；禁止二进制/加密字段进 log | pi 的 grepability 判据 | `scripts/` |
| 5 | `defineTool` 的**参数校验失败必须回填为 `tool_result(isError)`，禁止 throw 打断循环**（写成硬约束 + 单测） | pi/TypeBox | SDK_SURFACE §1、`plugin-tools` |
| 6 | 干预动作枚举化：`agent.interrupt()/compact()/rollback()/approve()/shutdown()` 统一走一个 Submission 通道，且每个都落 log | Codex `Op` 枚举 | `plugin-core` 的 `createReactAgent` 返回值 |
| 7 | 纵深三道闸：policy → fs/world 路径约束 → subprocess argv + 网络开关；passthrough 默认在 `doctor` 里标红 | Codex 四道闸（裁剪版） | ARCHITECTURE §3 的 `world` 插件组 |
| 8 | provider 传输降级路径：优先流式 WS，失败回退 SSE/HTTP，且降级落 log | Codex `FallbackToHttp` | `packages/providers/*` |
| 9 | `profiles/core` 断言：工具数 ≤ 8、system prompt token ≤ 1.2k，CI 失败即阻断 | pi 的三个赌注 | `bundles/`、`scripts/check-surface.ts` |
| 10 | 仓库根 `AGENTS.md` 写明"resist adding code to kernel"，配 `check-deps.ts` | Codex 治理 | 根目录、`scripts/` |
| 11 | 多 agent 并行探索的默认隔离单位 = **git worktree**（每分支独立 ledger + 独立工作树） | learn-claude-code `s12_worktree_task_isolation`、TiDB 演讲 | `plugin-teams`，SDK_SURFACE 的 C 组案例 |
| 12 | 知识进仓库：`AGENTS.md` / `skills/*.md` / `commands/*.md` 三类文件作为一等公民，可 git、可 review、可团队共享；装配时按需读盘 | pi 的学徒模式 + mattpocock/skills 主线 | `plugin-system-prompt`、`plugin-commands` |

> 1–3 是新增结构（目录/seam），需要改 `PROJECT_STRUCTURE.md` 与 `ARCHITECTURE.md`；4–12 是对既有文档的加严。

---

## 6. 明确不学的东西（拒绝清单）

抄思路不等于全盘接受。以下四项我们**明确不采纳**，并写下来防止以后被"成熟项目都这么做"说服：

1. **不学 Codex 用 Rust 重写内核。** 与"Bun + 类型安全 SDK"的定位正面冲突；我们只借它的"两通道 + 纵深 + 非穷尽协议"三条抽象。
2. **不学 pi 把进程内工具作为默认安全模型。** 坑二（克隆仓库即执行任意代码）不是可接受代价；`.agentic/tools/*.ts` 自动加载必须默认关闭，需显式 `trust` 才启用。
3. **不学 Claude Code 式的高频 prompt 变更。** Mario 的抱怨值得贴在墙上：*"9 点开始工作、工作流跑得好好的，10 点就坏了，下午 3 点又变成完全不同的行为——模型没变，变的是 harness。"* 我们的对策：prompt/schema 变化 = 稳定前缀变化 = 必须开新 `requestSeries`，且每次 release 在 changelog 里显式列出 prompt diff。
4. **不学"先给 17 个工具，之后再裁剪"。** 工具面是设计期决定。`repo-intel` 的 scout（8 个只读工具）与 analysis（17 个）分档就是这个原则的落实，不许合并成一个"全给"模式。
5. **不把长 prompt 当成弱模型的解药塞进 core。** 弱模型需要更细的工作流指令，那是 profile / `AGENTS.md` 层的事（pi 坑一的解法本身也说"自己动手写进 AGENTS.md"），core 保持素。

---

## 7. 需要你拍板的 4 个点

1. **`ctx.checkpoints` 一期是否必做？** 我的判断是**必做**——没有它，`fork`/分支探索/多 agent 并行都不成立（因为它们共享同一个真实世界）。代价是要维护影子 git，有磁盘与性能成本。你若认为一期只做单 agent 线性会话，可以推到二期，但 `parentId` 字段要一期就留好。
2. **session 树一期还是二期开？** 建议一期落 `parentId` 字段与 `fork` API，UI/CLI 的 `/tree` 导航放二期。
3. **三道闸的默认强度**：SDK 默认 `allow-all`（开箱能用，文档标红风险）还是默认 `workspace-write`（安全但要配置才能跑通，第一印象差）？我倾向后者 + `doctor` 明确提示如何放开。
4. **worktree 是否作为 teams 的默认隔离单位？** 好处是并行不互相污染；成本是对非 git 目录不适用、且需要处理 worktree 生命周期清理。

---

## 8. 原始资料

- **dsh**：`deepseek-ai/deepseek-harness` → `docs/architecture.md`（Cordis、profiles/bundles/patch、seam 三角色、turn/step、session log 不变量与 projection seam 的第一手来源）；社区解读（七层问题、inbox/jobs/goals、四大模块对应记忆-认知-行动-决策）。
- **pi**：`badlogic/pi-mono` → 现 `earendil-works/pi`，官网 pi.dev；Mario Zechner 演讲与博客 *What I learned building an opinionated and minimal coding agent*；Armin Ronacher *Pi: The Minimal Agent Within OpenClaw*；7 包结构（ai / agent / coding-agent / tui / web-ui / pods）；会话树命令 `/tree` `/fork` `/clone` `/compact` `/export` `/share`；社区对三个坑的总结。衍生项目：`can1357/oh-my-pi`（Rust 原生核心 + batteries included）。
- **Codex**：`openai/codex`（`codex-rs` 约百个 crate 的四层架构：Frontends / Kernel(core+protocol) / Capabilities(apply-patch, sandboxing, execpolicy, network-proxy, rollout, codex-mcp, prompts, skills…) / Foundation）；四大支柱（事件驱动 Actor 内核 / 单内核多前端 / 协议即契约 / 安全纵深防御）；`Op`、`EventMsg` 变体；`rollout` JSONL 与 `reverse_jsonl_scanner`；`approval_policy` × `sandbox_mode` 组合；为什么不做 RAG。
- **回退与分支**：pi 树形会话与其"回退不回退文件"的坑；Shadow Git 架构设计（git branch+tag 承载分支、`git checkout` 原子切工作目录、prune 防膨胀）；`shareAI-lab/learn-claude-code` 的 `s12_worktree_task_isolation`（教学版 harness 机制递进：loop → tool use → todo → subagent → skills → compact → 持久化 → 团队，"循环属于 agent，机制属于 harness"）；Arbor/Hypothesis-Tree（insight 回传让每次尝试加深理解，而非盲目 trial-and-error）；TiDB「薄 Agent Loop，厚 Control Plane」。

> 注：本文的"裁决"部分（§4）与"改动清单"（§5）是我基于以上材料对本项目做的选择，**不是任何一家的官方结论**。§7 四个问题需要你的判断。

---

## 9. 配套文档

- [ARCHITECTURE.md](./ARCHITECTURE.md) — 内核五原语、seam 清单、turn flow、session log
- [LOOP_ENGINEERING.md](./LOOP_ENGINEERING.md) — 循环设计与可控性（本文 L3/L6 的实现细节）
- [CONTEXT_BUDGET_MEMORY.md](./CONTEXT_BUDGET_MEMORY.md) — 阈值与记忆（本文 L1/L2 的成本面）
- [REPO_INTELLIGENCE.md](./REPO_INTELLIGENCE.md) — 代码理解层（本文 §4-3 的裁决对象）
- [SDK_SURFACE.md](./SDK_SURFACE.md) — tool 定义与三 surface 导入（本文 §5-5 的落点）
- [MCP_INTEGRATION.md](./MCP_INTEGRATION.md) — MCP 接入（本文 §4-4 的落点）
- [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) — 目录树（本文 §5-1/2/3 的落点）
