# Agent Harness 架构设计（基于 dsh / Cordis 思路的移植）

> 状态：**规划稿，待评审**。本文不含实现，只定义骨架、接缝（seam）、事件契约与目录归属。
> 调研对象：DeepSeek Harness（dsh）+ 其底座 Cordis（论文《A Programming Paradigm for Spatiotemporal Composability》）；第二参照系：pi（badlogic/pi-mono）、OpenAI Codex CLI、会话树/回退系（对比与取舍见 [HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md)）。
> 我们的项目：**Agent = Model + Harness**，对外是 SDK，入口保持 `createReactAgent()`。

---

## 0. 一句话对齐：dsh 的核心思路，以及我们要抄什么、不抄什么

dsh 的官方架构文档原话：

> Every part of the product is a plugin, including the model adapter, the tool registry, the session log, and the agent loop itself, so each is replaceable from configuration. There is no privileged core to patch.

关键差异不在"有没有插件"，而在**插件贡献什么**。Cordis 把插件收窄成三样东西，全部挂在一个共享上下文 `ctx` 上：

| 机制 | Cordis 的叫法 | 解决的问题 |
|---|---|---|
| 能力暴露 | **Service**（`ctx.provide` → `ctx.<key>`） | 空间可组合：插件之间只认 key，不 import 具体实现 |
| 扩展点 | **Typed Event**（`emit` / `waterfall` / `parallel` / `serial`） | 策略拦截：不改代码就能改写/拒绝一次请求 |
| 注册 = 可撤销 | **Reversible Effect**（`ctx.effect`） | 时间可组合：卸载时按逆序完整回滚，于是有热重载、测试隔离 |
| 依赖即装配顺序 | **Reactive Coeffect**（`ctx.inject`） | 依赖满足才激活；提供方变了自动重启消费方；不用手写启动顺序 |
| 生命周期 | **Fiber 状态机** PENDING→LOADING→ACTIVE→UNLOADING→DISPOSED（异常 FAILED） | 装配/拆解全程可观测 |
| 可替换能力 | **Seam** = Definition + Provider + Consumer 三角色 | "换一个实现"变成配置动作，而不是 fork |
| 组装配方 | **Profile**（具名组合）+ **Bundle**（分发单元）+ **Patch**（分层叠加） | 同一份代码，拼出完全不同的 Agent |
| 上下文真相源 | **Session Log**（append-only）+ `deriveMessages()` | "模型可见 = 已记录"，于是可 fork/续跑/审计/重放 |
| 执行分层 | **turn / step** 两级 | turn = 从领取输入到不欠任何后续工作；step = 一次模型请求 + 它触发的工具 |

**要抄的（这是 dsh 真正的价值）**：`ctx` + 三种贡献物、seam 三角色纪律、waterfall 做策略拦截、effect 可撤销、session log 作为唯一真相源、turn/step 分层、profile/bundle/patch 分层组装。

**不抄的**：Cordis 全套元理论（preservation / confluence / inertia 等）先不做形式化证明；Electron 桌面宿主、Web UI、ACP、Python SDK 双栈、session 格式迁移链，一期全部不做。

### 0.1 第二参照系：四个项目把"重量"压在不同地方

单看 dsh 会把架构想成唯一正解。把 pi / Codex / 回退系并排看，会发现它们各自把不确定性翻译成不同的工程制品，而**这四样互不冲突，可以叠加**：

| 参照 | 它的重心 | 本文采纳的部分 |
|---|---|---|
| dsh | 可撤销 + 可追溯 | 内核五原语、seam 三角色、session log 不变量（§2、§3、§5） |
| pi | 缩小能力面 + 校验回填 | 工具/prompt 预算受 CI 断言、参数校验失败不 throw、状态可 grep、知识进仓库 |
| Codex | 干预动作枚举化 + 边界纵深 | 两通道（Op/Event）抽象、三道闸、传输降级路径、"resist adding code to kernel"治理 |
| 回退系 | 时间轴 | `ctx.checkpoints`：**消息状态与世界状态一起回滚**（只回消息 = 分支不可比） |

它们的**互相打脸处**（工具面多大、prompt 多长、要不要索引、进程内工具 vs MCP、有无特权内核、默认安全强度）已在 HARNESS_CASE_STUDIES §4 逐条裁决，此处不重复。

**必须保守的地方**：dsh 是"一个产品 + 配置组装"，我们是**一个要对外发 SDK 的库**。SDK 的契约是 `import` 与类型，不是 YAML patch 行。所以我们做的是 **"微内核 + 插件"的 TS 库版**：配置组装能力保留（`defineProfile` + patch），但主路径是类型安全的 builder API。这是与 dsh 最大的有意偏离，见 §6。

---

## 1. 分层总览

```
┌──────────────────────────────────────────────────────────────┐
│  @agentic/core        createReactAgent()  ← SDK 门面（薄）      │
│                       只做「组装 profile → 起 ctx → 返回句柄」    │
├──────────────────────────────────────────────────────────────┤
│  Bundles 层           bundle-standard / bundle-minimal         │
│                       一组插件 + 一组 patch 的可分发打包          │
├──────────────────────────────────────────────────────────────┤
│  Plugins 层（能力提供者，全部可替换）                             │
│   plugin-agent-loop(react) plugin-llm plugin-tools plugin-mcp   │
│   plugin-session plugin-system-prompt plugin-context plugin-    │
│   plugin-memory plugin-sandbox plugin-policy plugin-observe     │
├──────────────────────────────────────────────────────────────┤
│  @agentic/kernel      ctx / Service / Event / Effect / Fiber    │  ← 微内核
│                       inject / plugin / defineSeam（唯一"特权"）  │     只有组合语义
└──────────────────────────────────────────────────────────────┘
```

内核里**没有 Agent 循环、没有工具、没有模型调用**。它只回答一个问题：多个能力如何安全地同时存在、并可被干净地拆掉。

---

## 2. 微内核 `@agentic/kernel`

### 2.1 五个原语（对外 API 面就这五个，刻意保持极小）

```ts
// 1) Context —— 服务容器 + 事件总线 + 副作用归属，一切的核心
interface AgentContext {
  /** 层级可见性：子 ctx 可读父 ctx 的全部服务与事件；父不可见子（子 Agent 隔离的基础） */
  fork(scope?: ScopeOptions): AgentContext;

  /** 提供服务：注册即 effect，插件卸载自动回收 */
  provide<T>(def: ServiceDefinition<T>, impl: T): Disposable;
  /** 消费服务：依赖满足才激活；提供方变化自动 deactivate → reactivate */
  inject<const K extends readonly string[]>(keys: K, cb: InjectedCallback<K>): Fiber;
  /** 唯一的副作用原语：一切注册最终都归结到它 */
  effect<T extends (ctx: AgentContext) => Dispose | undefined>(fn: T): void;
  /** 类型化事件，四种分发模式（见 §2.4） */
  on<E extends keyof EventMap>(name: E, mode: DispatchMode<E>, handler: Handler<E>): Disposable;
  emit<E>(name: E, payload: Payload<E>): EmitResult<E>;
  readonly parent?: AgentContext;
}

// 2) Plugin —— 一个模块只要导出 name + apply(ctx)
interface Plugin<P = object> {
  name: string;
  apply(ctx: AgentContext, options?: P): void | Promise<void>;
}

// 3) ServiceDefinition —— seam 的「声明」那一半
interface ServiceDefinition<T> { readonly key: string; readonly version: string }

// 4) Disposable —— 显式可撤销
interface Disposable { dispose(): void | Promise<void> }

// 5) defineSeam —— 强制三角色齐全（只有 Provider 不算 seam）
function defineSeam<D, P, C>(spec: {
  definition: ServiceDefinition<D>;
  provider: (ctx: AgentContext) => D;   // 可被 patch 替换
  consumer: (ctx: AgentContext) => C;   // 通常就是「模型可见的那个工具」
}): Seam<D, P, C>;
```

**内核铁律（写进 CI 校验）**：
1. 任何对 `ctx` 的变更必须经 `ctx.effect`（`provide` / `on` / `plugin` 都是它的特例）→ 结构上保证"凡注册必可撤销"。
2. 插件禁止 `import` 另一个插件的实现，只能 `inject` 服务 key。
3. 插件禁止自建不被内核追踪的资源（定时器、子进程、socket、文件句柄）→ 必须包进 `effect`。
4. 内核包（`kernel`）不允许依赖任何 plugin 包（依赖方向单向，用 `check-deps` 脚本守门）。

### 2.2 Fiber 与生命周期

```
PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED
              ↓ (throw)        ↑
            FAILED ────────────┘  (显式 reload)
```
配套生命周期事件：`plugin/ready`、`plugin/dispose`、`plugin/failed`、`plugin/fork`。装配与拆解全程可观测——这是后续做 `inspect` 命令和评测回归的地基。

### 2.3 响应式共效（这是 dsh 最容易被低估的一点）

`inject(['llm', 'tools'])` 声明后，运行时负责：
- 依赖齐了才 `apply`；
- `llm` 的提供方从 openai 换成 anthropic → 所有消费 `llm` 的插件自动重启，不需要重启进程；
- 加载顺序由服务依赖图推导，**没有任何插件硬编码"先启动谁"**。

对我们的直接收益：换模型、换工具后端、把执行环境从本地切到远程沙箱，都是"换一行配置"，而不是"改循环代码"。

### 2.4 四种事件分发模式（每个事件**必须**选定一种，且它是公开契约的一部分）

| 模式 | 语义 | 返回值 | 我们的用途 |
|---|---|---|---|
| `emit` | 按注册顺序观察，不等结果 | void | 观测类：`session/event`、`telemetry/*` |
| `waterfall` | 中间件链，`next()` 委托，**可短路** | 有 | **策略拦截核心**：`agent/pre-step`、`tools/pre-execute`、`llm/stream` |
| `serial` | 顺序执行，逐个等待 | 有 | 必须完成的初始化：`agent/created` |
| `parallel` | 并行执行 | void | 无副作用的广播：`turn/start` 通知 |

`waterfall` 是"不改内核也能改行为"的技术支点：监听者可以在自己拥有决策权时**直接返回而不调 `next()`**，下游只看到被改写后的结果。审批（human-in-the-loop）、Guardrail、参数重写、结果脱敏、重试，全部落在这里，而不是落在一套 `middleware` 数组上。

> 与上一版结构的差异：原 `packages/core/src/middleware/` 取消，中间件语义并入 waterfall 事件监听者。

---

## 3. Seam 清单（本项目最重要的设计交付物）

**规则：新增一项能力 = 设计齐三个角色。只写一个实现不算 seam。**

| Seam | `ctx` key | Definition（接口要点） | 默认 Provider | Consumer（模型可见面） |
|---|---|---|---|---|
| 模型接入 | `ctx.llm` | `send(req): AsyncIterable<StreamEvent>`、`capabilities` | openai-compatible | 循环内部（非工具） |
| 工具注册 | `ctx.tools` | `register(tool)` / `list(scope)` / `execute(call)` | 本地注册表 | `tools/*` → 工具 schema 进 prompt |
| Agent 循环 | `ctx.agentLoop` | `run(agent): AsyncIterable<AgentEvent>` | **ReAct loop** | `createReactAgent` |
| 会话日志 | `ctx.sessions` | `append(events)` / `deriveMessages()` / `open` / `fork(parentId)` | JSONL **树**（事件带 `parentId`）+ 内存索引 | 历史投影 |
| 检查点回滚 | `ctx.checkpoints` | `snapshot(scope)` / `rollback(to)` / `compare(a, b)` | 影子 git（世界状态）+ log 锚点（消息状态） | 被 `fork`/`rollback` 共享 |
| 提示词装配 | `ctx.systemPrompt` | `section(name, provider)` 有序拼装 | ReAct 模板（可覆盖） | system 消息 |
| 上下文工程 | `ctx.context` | `compact(state)` / `offload(ref)` | token 预算 + 摘要压缩 | 自动挂载于 `agent/request` |
| 记忆 | `ctx.memory` | `recall(q)` / `remember(items)` | in-memory | `memory_*` 工具 |
| 技能 | `ctx.skills` | `discover()` / `load(name)` / `trust` 分级 | 目录扫描（Agent Skills 规范） | L1 进 system prompt；**L2 由模型自己 `read_file`**（不新增工具） |
| 规划 | `ctx.planning` | `write(todos)` / `state` （**opt-in**，不入 core 面） | 内存 + log | `write_todos` 工具 |
| 文件系统 | `ctx.fs` | `ls/read/write/edit/glob/grep/delete?` + `capabilities` + `routes` | 本地 cwd 受限（virtual mode） | `ls`、`read_file`、`write_file`、`edit_file`、`glob`、`grep` |
| 子进程 | `ctx.subprocess` | `spawn(spec)` / `capabilities` | Bun 子进程 | `bash` 工具 |
| 沙箱 | `ctx.sandbox` | `wrap(argv): SpawnSpec` | passthrough（**默认在 `doctor` 里标红不安全**） | 被 fs/subprocess 共享 |
| MCP | `ctx.mcp` | `connect(spec)` / `listTools()` / `callTool()` | 官方 SDK client | `mcp__<server>__<tool>` |
| 策略 | `ctx.policy` | `decide(action): allow\|deny\|ask` | allow-all（开放可叠加） | waterfall 监听者 |
| 遥测 | `ctx.telemetry` | `trace/span/metric` | no-op | waterfall + emit 监听者 |
| 命令 | `ctx.commands` | `register(cmd)`（**不走模型 turn**） | 空 | CLI `/slash` |
| 代码理解 | `ctx.repoIntel` | `overview()` / `search()` / `trace()` / `impact()` | tree-sitter 图（或接外部 MCP） | `repo_overview`、`trace_path` 等 |
| 编排 | `ctx.workflow` | `defineWorkflow` → node 组合 | 在 `ctx.agentLoop` 之上 | workflow DSL（**不新建运行时**） |
| 输入收件箱 | `ctx.inbox` | `push(msg)` / `classify(): append\|newTask\|interrupt` | FIFO + 规则分类 | `turn/input-queued` 事件 |
| 协作 | `ctx.teams` | `roster` / `board` / `mailbox` / `budget-shard` / `worktree` | opt-in | 多 Agent 任务分发 |

**seam 的组合效应（照抄 dsh 最漂亮的一手）**：`fs`、`subprocess`、`sandbox` 共享**同一个"执行世界"**。把这三个 provider 一起指向远程沙箱，`bash`、`write_file`、PTY 会**整体搬过去**，不需要为每个工具各 fork 一份实现。所以：

> 定义 seam 时不能只看单个工具，要问"它和谁共享执行世界"。这决定了能不能一次替换、全局生效。

**四条来自案例研究与 Deep Agents 的加严**：
- **`checkpoints` 必须跨两个状态栈原子回滚**：消息投影 + 世界状态（fs / git / 外部副作用）。只回退消息是幻觉——pi 的 `/tree` 不回退文件，导致「方案 A / 方案 B 平行探索」其实共用一个真实世界，分支结果不可比。
- **fs / subprocess / sandbox / checkpoints 共享同一个「执行世界」**：把它们一起指向远程沙箱或影子仓库，能力整体搬迁，不为每个工具各 fork 一份。
- **`inbox` 是对「用户中途插话」这个不确定性的显式回答**：插话不能默默进消息数组，必须先分类（补充说明 / 新任务 / 打断）再落 log。

**第四条（来自 Deep Agents，工具面生成机制）**：**能力缺失 = 工具面缺失。** provider 必须声明 `capabilities`；工具声明 `requires`；装配阶段按能力过滤，**不支持的动作对模型直接隐藏，而不是返回一个权限错误**。模型看不见它做不到的事，就不会浪费一步去调它（详见 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §1.3）。同时与 profile 手工声明取并集：声明可读可审计，探测防漂移。

一期范围建议：`llm` / `tools` / `agentLoop` / `sessions` / `systemPrompt` / `mcp` / `policy` 七个必做；`checkpoints`（只做消息锚点 + 影子 git 快照，不含 compare）与 `inbox` 建议一并必做，否则 fork/resume 语义不完整。

**因为“基础工具不用手动定义”是你的硬要求，`fs` 与 `subprocess` 也从“定接口留空”提升为一期必做**（否则那 7 个内置工具跑不起来，见 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §2.1 与 §6-1/2）；`skills`（一期 L1/L2）与 `planning`（`write_todos`，opt-in）一并一期。定接口留空实现的只剩：`sandbox`（OS 级列二期）/ `memory` / `telemetry` / `repoIntel`。

---

## 4. 事件契约（Turn Flow）

沿用 dsh 的 turn/step 两级模型：**step = 一次模型请求 + 它触发的工具调用；turn = 从领取输入开始到不欠任何后续工作为止，含 0 个或多个 step。**

```
turn/start
  claim 下一条输入（+ 队列中的一条待注入消息）
  装配 system sections + tool schemas；投影 runtime context
  → agent/pre-step                [waterfall] reject | 改写 accepted messages
      拒绝 / 首次 enter 被改写成空 → 不开 step，直接结束 turn
      step/start
      agent/request               [waterfall] 解析实际路由（换模型/降级）
      prepareCall()  —— 取消发生在此阶段则 system 与 user 都不提交
      把被接受的 messages 作为 user/message 落盘
      从 session log 派生并冻结 model history
      llm/stream                  [waterfall] 可包一层重试/缓存/降级
        → agent/assistant-stream  [emit] start / chunk* / end（仅进程内，非 durable）
      tool/call*                  [durable]
      tools/pre-execute           [waterfall] 审批、参数重写、短路
      tools/execute               [waterfall] 真正执行（seam provider 在此）
      tools/post-execute          [waterfall] 截断、脱敏、结构化
      tool/result*                [durable]
      step/end
      工具还欠一次请求 / 有新输入到达 → claim → 下一个 step
  → agent/turn-stopping           [serial] 无 next()，通知收尾
turn/end
```

**durable（进 session log，可重放）与 live（进程内扩展点）必须严格分开**：
- durable：`turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`tool/call`、`tool/result`
- live：`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre-execute`、`tools/execute`、`tools/post-execute`、`agent/assistant-stream`、`agent/turn-stopping`

事件命名规范：`<domain>/<action>`，小写连字符，domain ∈ {`plugin`,`agent`,`turn`,`step`,`session`,`system`,`user`,`assistant`,`tool`,`tools`,`llm`,`fs`,`mcp`,`memory`,`policy`,`telemetry`,`command`}。

**新行为往哪放（对照表，评审重点）**：

| 目标 | 机制（不要改循环） |
|---|---|
| 加一个模型厂商 | 在 `ctx.llm` 注册 adapter |
| 加一个模型可见能力 | 在 `ctx.tools` 注册；schema 自动进 prompt 装配 |
| 拦截/审批一次工具调用 | 监听 `tools/pre-execute`（waterfall，可不调 `next()`） |
| 改写或拒绝用户输入 | 监听 `agent/pre-step` |
| 加模型可见上下文 | `agent.inject()`，落在下一个被接受的 request |
| 换执行环境（本地↔远程沙箱） | 同时换 `fs` + `subprocess` + `sandbox` 的 provider |
| 让某个会话用另一套能力集 | 组装一个 profile / preset（不是 fork 代码） |
| 加一条人工命令（不过模型） | 在 `ctx.commands` 注册 |

---

## 5. Session Log 作为唯一真相源

dsh 的原文约束值得逐字采纳：**Model-visible means logged** —— 任何模型能看到的输入，必须先是一条 session event；运行时用不变量校验"模型请求可从 log 重建"。

我们的落地版本（一期简化，但方向不变）：

1. `SessionEvent` 为 append-only 联合类型；`deriveMessages(sessionId)` 是模型历史的**唯一**投影入口。
2. 插件若要改写已有消息内容，**不许直接改 log**，只能注册 **pure message projection**（纯函数 `events -> messages`，由 `ctx.sessionProjections` 收纳）。这条约束保证 log 不可变 + 可重放 + 可 fork。
3. `turn/start`、`turn/end`、`step/end` 是 fork / resume / 断点续跑的边界锚点。
4. 一期存储：`session.v0.jsonl`（内存索引 + Bun 文件写）。压缩（zstd）、迁移链（vN→vN+1）列为二期，**但版本号从 v0 就开始写**，避免以后补不了。
5. 失败 step 必须补写 `tool/result` 占位（missing result），否则历史不可重放——dsh 踩过这个坑。
6. **可 grep 性是验收项，不是审美**：会话文件必须能被 `grep` / `jq` 直接读懂；禁止任何「模型可见内容只存在于压缩或二进制表示里」。dsh / pi / Codex 三家不约而同选 JSONL 明文，这是「唯一真相源 + 纯投影」的必然结果，不是巧合。由 `scripts/check-grepability.ts` 守门。
7. **线性改树**：每个事件带可选 `parentId`，`fork` = 在某个节点挂新子链，`revise` = 追加修订而非原地改。一期先把字段和 `fork` API 落好（成本极低），`/tree` 导航与 `compare` 放二期——事后补 `parentId` 会让历史 v0 数据不可分支。

```ts
type SessionEvent =
  | { type: 'turn/start';      turnId: string; at: number }
  | { type: 'user/message';    turnId: string; content: ContentPart[] }
  | { type: 'system/message';  turnId: string; content: string; revision: number }
  | { type: 'step/start';      stepId: string; requestId: string }
  | { type: 'assistant/message'; stepId: string; content: ContentPart[]; toolCalls: ToolCall[] }
  | { type: 'tool/call';       stepId: string; callId: string; name: string; args: unknown }
  | { type: 'tool/result';     stepId: string; callId: string; result: ToolResult }
  | { type: 'step/end';        stepId: string; usage: Usage; finish: FinishReason }
  | { type: 'turn/end';        turnId: string }
```

---

## 6. Profile / Bundle：SDK 版的组装机制（与 dsh 的有意偏离）

dsh 用 `cordis.patch.yml` 分层叠加：bundle（按 profile 声明顺序）→ profile patch → home patch → `--patch`，并可用 `dsh --profile web --dump-config` 打印实际组装出的整棵树，**其中每一行都能被你自己的 patch 替换**。

我们保留这个"分层可覆写 + 可 dump"的思想，但换成 TS 类型安全的载体：

```ts
// 一个 Bundle = 一组插件 + 它们的配置行（可分发单元）
import { defineBundle } from '@agentic/kernel';
export const bundleStandard = defineBundle({
  name: '@agentic/bundle-standard',
  plugins: [pluginSession, pluginLlm, pluginTools, pluginAgentLoopReact, pluginSystemPrompt],
  // 声明式配置行，可被上层覆写，不是直接 mount
  rows: [{ id: 'llm', use: 'openai', with: { model: 'gpt-4o' } }],
});

// 一个 Profile = 具名配方：按序叠 bundle，再叠自己的 patch
const myProfile = defineProfile({
  name: 'review-bot',
  extends: ['standard'],
  patches: [
    { id: 'llm', use: 'anthropic', with: { model: 'claude-...' } },  // 整行替换
    { id: 'tools.builtin', add: ['read_file', 'bash'] },
  ],
});

// SDK 门面：createReactAgent 就是「sdk profile + react loop」的薄封装
const agent = createReactAgent({ profile: myProfile, prompt: REACT_OVERRIDE });
```

内置 Profile 模板（对齐 dsh 的 web/headless/sdk/sdk-minimal/acp）：

| Profile | 形态 | 场景 |
|---|---|---|
| `sdk` | 标准 bundle + mcp + policy | SDK 默认，绝大多数接入 |
| `sdk-minimal` | 单 bundle 自带完整显式树，不含标准 bundle | 冷启动最快、体积敏感、嵌入式 |
| `cli` | `sdk` + 交互循环 + commands | 本地手工验证 |
| `headless` | `sdk` + one-shot runner，不起 server | CI、脚本 |

**必须保留 dsh 的一个诊断能力**：`agent.dumpConfig()` 打印本机实际组装出的插件树与每一行配置——用户能定位"到底是哪层把 llm 改了"。SDK 没有 YAML 分层，但没有 dump 就无法排障，这个不能省。

---

## 7. createReactAgent 的定位（保持你最初的硬要求）

`createReactAgent()` 是**唯一**的 Agent 创建入口，但它必须**薄**：只做「解析 profile → 创建根 ctx → 装配 fiber 树 → 返回 Agent 句柄」，一行循环逻辑都不写。循环在 `plugin-agent-loop(react)` 里，作为 `ctx.agentLoop` 的默认 provider。

```ts
const agent = createReactAgent({
  profile,                 // 配方；不传则用 sdk
  model: 'openai:gpt-4o',  // 便捷糖，等价于 patch llm 行
  tools: [myTool],         // 注册进 ctx.tools
  mcp: { servers: 'agent.mcp.json' },  // 见 MCP 文档
  prompt?: ReactPromptSpec,             // ReAct 模板可整体覆盖（开放问题 §9-4）
  maxSteps: 16,            // turn 内的 step 上限
  on?: { 'tools/pre-execute': approvalListener },  // 直接挂 waterfall 监听
});

await agent.run(input);                    // 一次性
for await (const ev of agent.stream(input)) {}  // 流式（durable + live 混合投影）
await agent.fork(atTurnBoundary);          // 从 log 派生
await agent.resume(sessionId);             // 断点续跑
agent.dispose();                           // 全树 effect 逆序回滚（不重启进程）
```

`agent.dispose()` 的可撤销性来自 §2.1 铁律 3——这是"一切皆插件"在 SDK 场景下最实际的回报：单测里可以真起真卸一个带 MCP 子进程的 Agent，不留僵尸进程。

---

## 8. 目录归属（配套 PROJECT_STRUCTURE.md，此处只给映射）

```
packages/
  kernel/            @agentic/kernel            §2 五原语 + fiber + dispatch
  types/             @agentic/types             纯类型（Seam 接口、SessionEvent、ToolDefinition）
  core/              @agentic/core              createReactAgent + defineProfile/defineBundle + dump
  plugins/
    agent-loop-react/  plugin-llm/  plugin-tools/  plugin-session/
    system-prompt/     context/     memory/         fs/ subprocess/ sandbox/
    mcp/               policy/      observability/  commands/
  bundles/           bundle-standard/  bundle-minimal/
  providers/         provider-openai/ provider-anthropic/ provider-ollama/  （llm seam 的实现）
  cli/               @agentic/cli     profile 启动器 + dump-config + inspect
```
依赖方向（单向，CI 校验）：`cli/core/bundles → plugins → types ← kernel`；`plugins` 之间**零依赖**，只通过 `ctx` key 协作。

---

## 9. 开放问题（请逐条拍板）

1. **内核自研 vs 直接用 Cordis**：Cordis 源自 Koishi 生态、dsh 是 vendored 进仓库的。我们自研一个约 800 行的 TS 精简内核（只做 effect/inject/service/event 四件事）可控性更好，但偏离官方；直接用 Cordis 省一大块，代价是多一个上游依赖 + 概念耦合。**建议自研精简内核。**
2. **seam 数量一期收敛到 7 个**是否够（§3 建议清单）？如果你一期就想做代码类 Agent，`fs/subprocess/sandbox` 必须提前。
3. **profile 是否对外暴露**：暴露则用户可自定义配方（强但概念多），不暴露则只有 `createReactAgent` 选项对象（简单但失去分层覆写）。**建议一期内部实现分层、对外只暴露 `patches` 数组。**
4. **ReAct prompt 模板是否允许整体覆盖**：允许则 seam 更纯，不允许则可保证行为一致。dsh 的答案是允许（`ctx.systemPrompt` 分段装配）。
5. **Session log 一期是否必做**：不做则没有 fork/resume/审计，也失去了"模型可见=已记录"这条最有价值的不变量。做了才有断点续跑。**强烈建议一期必做（JSONL 即可）。**
6. **waterfall 与"中间件"心智冲突**：SDK 用户可能更熟 `use(middleware)`。是否在外层提供 `agent.use()` 语法糖，内部转成 waterfall 监听者？**建议加糖，内核不变。**
7. **`@agentic/*` scope 是否与 npm 现有包冲突**（上一版遗留，发包前必须确认）。
8. **新增的 4 个取舍**（`ctx.checkpoints` 是否一期必做、session 树一期开多少、三道闸的默认强度、worktree 是否作为 teams 默认隔离单位）见 [HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md) §7。

---

## 10. 配套文档

- 目录结构与包规划：[PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)
- MCP 接入与管理方案：[MCP_INTEGRATION.md](./MCP_INTEGRATION.md)
- **Agent Loop 设计与可控性（教程）**：[LOOP_ENGINEERING.md](./LOOP_ENGINEERING.md)
- **阈值 / Context 关联 / Memory（教程）**：[CONTEXT_BUDGET_MEMORY.md](./CONTEXT_BUDGET_MEMORY.md)
- **代码理解层（为何不再只读 README）**：[REPO_INTELLIGENCE.md](./REPO_INTELLIGENCE.md)
- **Tools 定义与三 surface 导入路径规范**：[SDK_SURFACE.md](./SDK_SURFACE.md)
- **成熟 harness 案例研究与取舍裁决（dsh / pi / Codex / 回退系）**：[HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md)
- **内置原子工具与 Skills（Deep Agents 调研 + 工具格式契约）**：[BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md)
