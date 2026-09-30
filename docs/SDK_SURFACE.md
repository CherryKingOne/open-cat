# SDK 对外契约：Tools 定义、导入路径规范与案例规划

> 回答你的三点：**Tools 用 function calling 还是别的方式定义？Agent / Workflow 各自的专业导入路径怎么分？多 Agent 与 Workflow 案例给哪些？**
> （内置工具开箱可用 / Skills / 写死 vs 继承 / 工具格式定稿 —— 这四问在 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md)，本文只保留与之一致的契约面）
> 配套：[ARCHITECTURE.md](./ARCHITECTURE.md) §3 seam、[LOOP_ENGINEERING.md](./LOOP_ENGINEERING.md)、[REPO_INTELLIGENCE.md](./REPO_INTELLIGENCE.md)、[HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md)、[SANDBOX.md](./SANDBOX.md)

---

## Part 1 Tools 怎么定义

### 1.1 先破除一个误解

"定义 tool" 不等于"写一份 JSON Schema 给模型"。一个可上线的 tool 定义要同时回答**五个问题**，缺一个都会在生产里出问题：

> 五个**维度**是关注点清单；逐字段的必填/建议定稿（12 个字段 + `ToolResult` 联合类型）在 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §5.1，**那里是唯一真相源**，不两处维护。

| 维度 | 内容 | 缺失后果 |
|---|---|---|
| **契约** | name + description + 参数 schema（JSON Schema） | 模型选错/传错参数 |
| **执行** | `run(args, ctx, signal)` → 结构化结果 | — |
| **风险** | 只读？破坏性？触达外部世界？是否幂等 | 无法自动决策"要不要审批" |
| **失败语义** | 错误怎么回给模型（**必须可行动**） | 模型瞎重试，撞满 maxSteps |
| **成本** | 预估 token / 时长 / 费用 / 是否并发安全 | 预算层无法提前拦（见 CONTEXT_BUDGET_MEMORY §1） |

```ts
export const searchOrders = defineTool({
  name: 'search_orders',
  description: '按客户 ID 或时间范围查询订单，返回订单号与状态。单次最多 50 条。',
  parameters: {                              // JSON Schema，strict 模式（见 §1.3）
    type: 'object',
    additionalProperties: false,
    required: ['customerId'],
    properties: {
      customerId: { type: 'string' },
      since:      { type: 'string', format: 'date' },
      limit:      { type: 'integer', minimum: 1, maximum: 50, default: 20 },
    },
  },
  annotations: {                             // ★ 喂给 policy 的现成语义（对齐 MCP annotations）
    readOnly: true, destructive: false, openWorld: false, idempotent: true,
  },
  cost: { estTokens: 600, estLatencyMs: 800, concurrentSafe: true },
  validate(args) { return OrderQuery.parse(args); },   // 前置校验：在扣钱之前拦住
  async run(args, ctx, signal) {
    const rows = await db.orders.query(args, { signal });
    return ok(rows.map(toCompactOrder));               // ★ 结构化，别塞原始 HTML/JSON 全文
  },
  describeError(e) {                                 // ★ 把错误翻译成模型能行动的指令
    if (e.code === 'CUSTOMER_NOT_FOUND') return '客户 ID 不存在，请先调用 list_customers 获取有效 ID';
    if (e.code === 'TIMEOUT')            return '查询超时，请缩小时间范围或减少 limit 后重试';
    return `不可恢复错误：${e.message}`;               // 明确不可重试，阻断瞎重试
  },
});
```

> **`describeError` 是我特别想让你加上的一个字段。** 工具报错时把原始 stack trace 甩回给模型，是"Agent 反复重试同一件事"的头号成因。给一句**可行动的下一步**，循环就能自己爬出来。

### 1.2 五种"给 Agent 加能力"的通道（不止 function calling）

| 通道 | 机制 | 适用 | 我们的落点 |
|---|---|---|---|
| **A. Function calling**（原生 tool use） | 模型输出结构化 `tool_calls`，harness 执行 | 离散、参数明确、需审计的能力 | `ctx.tools` 主路径，**默认选择** |
| **B. MCP** | 外部 server 提供 tools/resources/prompts | 第三方集成、跨语言、非自己维护的能力 | `plugin-mcp` → 同样注册进 `ctx.tools` |
| **C. Code-as-action**（让模型写代码来编排能力） | 模型在沙箱里写 TS/Python，调用 SDK 而非逐个 tool | 循环/批量/数据变换类任务——用 function calling 表达会退化成"几十次调用" | `plugin-sandbox` + `run_script` 单工具。**沙箱提到一期后这条路提前有条件地可行**：只在 `native`/`container` 后端就位时上面上，`sandbox: 'off'` 时该工具**不在模型面上**（见 SANDBOX.md §5.2 / §7） |
| **D. Commands**（人工触发，**不过模型 turn**） | 注册进 `ctx.commands`，dispatch 不产生 turn | 明确操作、零不确定性、省钱 | `/deploy`、MCP prompts 映射 |
| **E. Skills**（纯提示型能力包，**零代码**） | 目录 + `SKILL.md`：L1 元数据进 system prompt，L2 由模型自己 `read_file` 读正文 | 方法论 / 流程 / 领域知识；"要新知识不要新逻辑" | `ctx.skills`（`plugin-skills`）——**不新增机制，L2 就是内置的 `read_file`** |

**选型判据（一句话版）**：
> 需要模型**决策**的用 A/B；需要模型**计算/编排**的用 C；不需要决策的用 D；只是要给它**一套做事的方法**用 E。
> 常见错误是把 D 类操作也做成 tool（每次都要模型想一想，白花一次请求）；以及把 C 类任务硬拆成 20 个 tool（每步一次往返，又慢又贵又易错）。
> **E 与 A 的分界**：能靠"写清楚步骤"解决的，绝不要写成工具——技能是 Markdown，进 git、可 review、不动代码就能改；一旦写成 `defineTool`，它就变成了一个需要维护、带版本、占工具面预算的**能力**。用户自定义工具三条路的完整判据见 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §3.5。

**A 与 B 的分界（补判据，来自 pi 的教训）**：
> 同机同进程 + 代码自己可信 → **直接 `defineTool` 一个文件**（`.agentic/tools/*.ts`），不要为了"看起来规范"先包成 MCP server——没有子进程、没有 JSON-RPC 握手、没有序列化损耗。pi 的默认四个工具全是进程内的。
> 跨语言 / 第三方维护 / 需要进程隔离 / 要复用给别人的 agent → **MCP**。
> ⚠️ 进程内工具的权限等同当前进程，因此：从外部仓库自动加载工具文件必须**默认关闭**，显式 `trust` 才启用（见 HARNESS_CASE_STUDIES §6-2）。

**工具面数量是设计期的决定，不是"接进来以后再说"。** pi 立项的直接依据：Terminal Bench 上的 Terminus 只给模型**一个工具**（向 tmux 发按键、读 ANSI 序列），却常年前三。今天的前沿模型已经被 RL 训练得天然知道 harness 是什么，**可选项越少，行为越可预测**。本项目的硬约束：`core` profile ≤ 8 个工具，超出即 `check-surface.ts` 失败。

### 1.3 四条工程纪律

1. **strict schema + `additionalProperties: false` + 前置校验。** 参数校验失败应该在**发出请求前**拦住，不是等模型被 provider 报错打回。
2. **★ 校验失败必须回填，禁止 throw。** 非法参数要变成一条 `tool_result(isError: true, "字段 limit 必须是 1–50 的整数，收到 500")`，让模型自己改；抛出异常会直接打断整个循环。**这是自己手写 agent 最常见的死法（一次参数幻觉 → 整个会话崩）**；pi 用 TypeBox 的“一份定义三产出”（TS 类型 / JSON Schema / 运行时校验器）把它堵住了。必须有单测覆盖。
3. **结果必须结构化且可截断。** 返回 `{ok, data, next?}`，超过阈值走 artifact 卸载（见 CONTEXT_BUDGET_MEMORY §2.3）——**工具原始输出直接灌进上下文，是上下文爆炸的主要来源**。
4. **工具名与描述是提示词的一部分。** 描述要写清"何时用 + 何时**不**该用 + 上限"。`tools/pre-execute` waterfall 上还可以做 schema 动态改写（对特定租户隐藏字段）。

### 1.4 工具多了怎么办（与 MCP 同一套解法）

> 3–7 个高价值工具胜过 50 个玩具工具；**工具越多，模型选错率越高**（成熟代码索引实现把 17 个工具裁成侦察模式 8 个，理由就是这条）。

三层解法（一期做前两层）：
- **作用域注册（scoped registry）**：不同 profile / 不同 agent 只挂自己需要的工具集；子 agent 用 `agent.ctx` 做作用域注册（dsh 的原语）。
- **渐进披露**：超阈值时只暴露 `search_tools` + `call_tool` 元工具（与 MCP 方案统一实现）。
- **Tool Router**：`tools/pre-execute` 上按意图动态筛选/重排候选工具（二期）。

---

## Part 2 导入路径规范（三个 Surface，各自独立入口）

你强调的点：**Agent 有 Agent 的专业导入路径，Workflow 有 Workflow 的专业导入路径。** 我把对外契约定成**三个 surface**，互不混用：

```
① Agent surface     —— 模型自主决策的循环（ReAct）
② Workflow surface  —— 人预先定义路径，节点里可以放 agent
③ Team surface      —— 多 Agent 协作（roster / 任务板 / mailbox / handoff）
```

### 2.1 规范一览（这是 SDK 的门面设计，评审重点）

```ts
// ── ① Agent surface ─────────────────────────────────────────
import { createReactAgent, defineProfile, defineBundle } from '@agentic/core';
import type { AgentHandle, AgentEvent, AgentState }        from '@agentic/types';
import { defineTool, ok, fail }                           from '@agentic/plugin-tools';   // T1：数据 + 函数
import { withGuards, withRetry, withCache }                from '@agentic/plugin-tools/operators'; // T2：装饰而非继承
import type { FsBackend, ShellBackend, LlmAdapter }        from '@agentic/types';          // T2：实现 interface
// 内置工具 ls/read_file/write_file/edit_file/glob/grep/bash(/task) **不需要 import**，由 profile 装配
import { openai }                                          from '@agentic/provider-openai';   // 或 '@agentic/plugin-llm/openai'

// ── 沙箱（既可给 agent 用，也可脱离 agent 自己当执行环境）───────
import { nativeSandbox, defineSandboxPolicy }               from '@agentic/sandbox';
import { containerSandbox }                                 from '@agentic/sandbox/container';
import { createSandbox }                                    from '@agentic/sandbox';       // ★ 单独用：await using sbx = await createSandbox(...)
import { fakeSandbox }                                      from '@agentic/sandbox/testing'; // 单测里替掉真笼子

// ── ② Workflow surface ──────────────────────────────────────
import { defineWorkflow, sequence, parallel, branch, loop, step } from '@agentic/workflow';
import { human, checkpoint, retry, compensate }                   from '@agentic/workflow/operators';
import { agentNode }                                              from '@agentic/workflow/nodes';  // ← 把 agent 作为节点

// ── ③ Team surface ──────────────────────────────────────────
import { defineTeam, supervisor, pipeline, handoff }  from '@agentic/teams';
import { taskBoard, mailbox, sharedBlackboard }        from '@agentic/teams/coordination';
import type { TeamEvent, AgentRole }                   from '@agentic/types/teams';

// ── 禁止的写法（CI 拦截）─────────────────────────────────────
import { internals } from '@agentic/core/src/internal/xxx';   // ✗ 深路径
import { ReactAgentImpl } from '@agentic/plugin-agent-loop-react/internal'; // ✗
```

**命名动词规范（对外 API 一致性，比结构更重要）：**

| 前缀 | 含义 | 例 |
|---|---|---|
| `createX()` | 工厂：返回**可运行实例**（有副作用、要 dispose） | `createReactAgent`、`createContext` |
| `defineX()` | 声明：返回**不可变描述对象**（无副作用） | `defineTool`、`defineProfile`、`defineBundle`、`defineWorkflow`、`defineTeam` |
| `withX()` | 装饰：返回加了能力的新对象 | `withRetry`、`withBudget` |
| `on(event, handler)` | 订阅：返回 `Disposable` | 所有扩展点 |
| 名词复数导出（`sequence`/`parallel`/`branch`） | 组合子，可嵌套 | workflow operators |

> **`create` vs `define` 的区分不是糖衣，是内存与安全语义**：`define*` 必须无副作用、可复用、可跨进程序列化；`create*` 拥有资源生命周期，必须配 `dispose()`。SDK 用户靠这两个词就能判断"要不要清理"。这是对外开放的库最容易失守的地方。

### 2.2 包的 exports 矩阵（每个包对外承诺什么）

| 包 | 根入口 | 子路径 |
|---|---|---|
| `@agentic/core` | `createReactAgent` `defineProfile` `defineBundle` | `/testing`（mock provider，供用户写测试） |
| `@agentic/types` | 全类型 | `/teams` `/workflow` `/session` |
| `@agentic/kernel` | `createContext` `defineSeam` `Service` | `/testing` |
| `@agentic/workflow` | `defineWorkflow` + 组合子 | `/operators`（retry/human/checkpoint/compensate）、`/nodes`（agentNode/toolNode/llmNode） |
| `@agentic/teams` | `defineTeam` `supervisor` `pipeline` `handoff` | `/coordination`（taskBoard/mailbox/blackboard） |
| `@agentic/plugin-*` | 各自 `pluginX` + 该能力的公共类型 | 通常无子路径（保持"一个能力一个包"的纯粹性）；**例外：`@agentic/plugin-tools/operators`（`with*` 装饰器）** | 
| `@agentic/plugin-tools` | `defineTool` `ok` `fail` `ToolResult` | `/operators`（withGuards/withRetry/withCache）——★ 这是"不做基类但保留少写重复"的唯一对外装饰面 |
| `@agentic/plugin-fs` | `pluginFs` + 内置文件工具装配 | `/backends`（`StateFs`/`LocalFs`，供用户实现 `FsBackend` 时复用受限模式） |
| `@agentic/sandbox` | ★ `SandboxBackend` 契约 + `defineSandboxPolicy` + `probe()` + `createSandbox` | `/native`（Seatbelt/bwrap/Win）、`/container`（docker/podman）、`/remote`（e2b/daytona/modal，二期）、`/iso`（CoW 工作区视图，二期）、`/testing`（`fakeSandbox`） |
| `@agentic/plugin-sandbox` | `pluginSandbox`：ctx.sandbox 装配 + **派生 `ctx.fs` / `ctx.subprocess`** + fail-closed | — |
| `@agentic/provider-*` | 厂商 provider | — |
| `@agentic/cli` | bin | — |

**Workflow / Teams 为什么可以是"薄包"而不是新运行时**（关键设计决定）：
> 它们不新增执行引擎。`plugin-workflow` 与 `plugin-teams` 只是**在 `ctx.agentLoop` 之上提供编排 provider**：workflow 的每个 node = 一个作用域化的 step/turn；`parallel` = 并发 turn + 预算分片；`human` = `awaiting-approval` 状态 + 持久化 checkpoint。所以它们天然共享 session log、事件、预算、可撤销性。
> **反过来做（给 workflow 单独一个 runtime）会得到两套日志、两套预算、两套中断语义** —— 这是很多框架在生产化的转折点崩掉的原因。

### 2.3 Workflow vs Agent：什么时候用哪个（决策表）

| 判据 | 用 Workflow | 用 Agent |
|---|---|---|
| 路径能否穷举 | ✅ 能 | ❌ 不能/不值得穷举 |
| 步骤数 | 固定或半固定 | 运行时才知道 |
| 失败处理 | 每种失败都有预案 | 需要现场判断 |
| 成本可预测性 | 高（可预算到单步） | 低（靠 maxSteps/时间预算兜） |
| 审计要求 | 强合规流程 | 探索型任务 |

> 行业共识（也是 OpenAI 早期对 agent/workflow 的区分）：**大多数号称 Agent 的系统其实是 workflow。** 这不是贬义——**能用 workflow 就别用 Agent**，路径确定的东西交给模型自主决策只会更贵更不稳。真正的分工是：**workflow 定骨架，agent 填进骨架里那些"不确定"的节点**（`agentNode`）。Dots 的形态（人交给它一项**责任**，它自己拆解推进，只在需要拍板时回来问）则说明：agent 适合"目标给定、路径未知"的长任务，而它的可控性完全靠 harness 的权限规则与预算机制，不靠模型自觉。

---

## Part 3 案例规划（`examples/` 是你的第二份文档）

SDK 的真实体验在示例里。规划 8 组（A 根 / B 可控 / C 多 Agent / D Workflow / E 项目理解 / F 扩展 / **G 内置工具与技能** / **H 沙箱**），**每组都从"最小可跑"到"有生产语义"**，并明确各自演示哪条设计纪律。

### A. Agent 基础（6 个）

| # | 案例 | 演示要点 |
|---|---|---|
| A1 | `01-minimal-react-agent` | `createReactAgent().run()`；一次 turn 一个 step |
| A2 | `02-define-tool` | 五维 tool 定义（含 annotations/cost/describeError） |
| A3 | `03-streaming` | durable + live 事件混合投影；`for await` 消费 |
| A4 | `04-budget-degrade` | 四维预算撞线 → **受控降级**（partial + resumeToken），而非截断输出 |
| A5 | `05-resume-from-checkpoint` | session log fork/resume；副作用幂等 journal |
| A6 | `06-replace-provider` | patch 一行换模型；验证消费方**自动重启**（不重启进程） |

### B. 可控性（4 个，对应 LOOP_ENGINEERING）

| # | 案例 | 演示要点 |
|---|---|---|
| B1 | `10-approval-waterfall` | 监听 `tools/pre-execute`，破坏性动作 → `awaiting-approval`（可恢复状态，不是失败） |
| B2 | `11-loop-detection` | 参数规范化哈希 + 进展指标 → 连续 2 轮无进展触发 replan |
| B3 | `12-rate-limited-provider` | **假 provider**：RPM 型 429 → 退避+jitter；TPM 型 429 → 降 token/换档位；额度耗尽 → 停止。**这个案例同时是回归测试资产** |
| B4 | `13-scanner-always-on` | 常驻形态：Scheduled Scan 外部唤醒 + **自动去重** + Activity View（对齐 Dots） |

### C. 多 Agent（4 个，对应 `@agentic/teams`）

| # | 案例 | 拓扑 | 关键难点 |
|---|---|---|---|
| C1 | `20-supervisor-worker` | Supervisor 派发 → worker 执行 | 上下文隔离（子 ctx 见父、父不见子）；worker 结果如何回灌 |
| C2 | `21-pipeline-specialists` | Research → Draft → Review 串行交接 | **交接契约**：结构化工件而非自然语言接力 |
| C3 | `22-handoff-routing` | 分诊 agent 把会话 **handoff** 给专家 | 会话所有权转移 + 历史裁剪 + 预算随迁 |
| C4 | `23-parallel-fanout-board` | 队长 + 任务板 + mailbox，并行子 agent（带依赖） | **并发预算分片**（否则子 agent 一起撞 TPM 墙）；失败隔离；死锁检测 |

> C4 要重点体现一条：**多 Agent 的成本与限流是"共享池"问题，不是各自独立问题。** 子 agent 必须从父预算**分片**（dsh 里子 agent 用 `agent.ctx` 做作用域注册，同一思路），否则你给父设了 `maxSteps`，10 个子各跑 50 步照样爆账。这也是"并行 fan-out 看起来很美、上线就被限流打回"的直接原因。

### D. Workflow（5 个，对应 `@agentic/workflow`）

| # | 案例 | 演示要点 |
|---|---|---|
| D1 | `30-sequence-basic` | `sequence(step(a), step(b))`；每步一个 checkpoint |
| D2 | `31-branch-and-loop` | 条件边 + 受限循环（循环上限即 step 预算） |
| D3 | `32-parallel-with-join` | fan-out/join、失败策略（`fail-fast` vs `best-effort`）、并发闸 |
| D4 | `33-human-in-the-loop` | `human()` 节点：暂停 → 落盘 → 进程可退出 → 数天后继续（真正的持久化中断） |
| D5 | `34-agent-node-mixed` | **workflow 骨架 + `agentNode` 填不确定节点**（推荐生产形态） |
| D6 | `35-saga-compensation` | 补偿事务：副作用序列失败后逐步回滚（对齐 §幂等 journal） |

### E. 项目理解（2 个，对应 REPO_INTELLIGENCE）

| # | 案例 | 演示要点 |
|---|---|---|
| E1 | `40-repo-overview` | `repo_overview` 四阶段产物；`intentVsReality` 揪出 README 与现实不一致 |
| E2 | `41-orient-execute-validate` | 三阶段门禁：只读 scout → 计划审批 → 执行 → 验证；图优先纪律 |

### F. 扩展（2 个，对应"一切皆插件"的核心卖点）

| # | 案例 | 演示要点 |
|---|---|---|
| F1 | `50-write-a-plugin` | 30 行写一个 provider 替换某能力，**不 fork 任何代码** |
| F2 | `51-effect-cleanup` | `agent.dispose()` 逆序回滚：MCP 子进程、定时器、监听器全回收（无僵尸进程） |

### G. 内置工具与技能（7 个，对应 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md)）

| # | 案例 | 演示要点 |
|---|---|---|
| G1 | `60-zero-config-builtin` | `createReactAgent().run('把 logs/ 里的 error 汇总')`——**不传一个 tool**，拿到 7 个原子工具（+core profile 再给 `task`）就能读/改仓库 |
| G2 | `61-capability-hiding` | ★ backend 不支持 `delete` → `delete` **从请求时的 schema 里消失**（而不是调了才报错）；`scout` 只读面下写工具同样不在面上 |
| G3 | `62-artifact-offload` | 大结果 → `/artifacts/` → 模型自己 `read_file(offset, limit)` 读回；验证内部路径与项目路径隔离 |
| G4 | `63-skill-progressive` | L1 目录 → L2 `read_file` → L3 资源；+ `skills/activated` durable 事件（命中可评测） |
| G5 | `64-custom-tool-in-process` | `.agentic/tools/*.ts` 自动加载 + **默认关闭需 `trust`**（pi 的坑二） |
| G6 | `65-wrap-a-backend` | ★ 用 `withGuards` / 实现 `FsBackend` 包装而非继承基类（T2 扩展点） |
| G7 | `66-edit-file-failure` | 失败可行动：`NOT_UNIQUE` / `NOT_FOUND` 的回填文本→模型自己改，**全程无 throw** |

### H. 沙箱（7 个，对应 [SANDBOX.md](./SANDBOX.md)）

| # | 案例 | 演示要点 |
|---|---|---|
| H1 | `70-native-workspace-write` | 一行声明笼子：`sandbox: 'native:workspace-write'`——bash 能写工作区但写不了宿主 |
| H2 | `71-deny-is-absence` | ★ deny `~/.ssh` → 目录直接"缺席"（tmpfs / 不 bind）而不是 EACCES；**`read_file` 与 `cat` 给出同一个答案** |
| H3 | `72-capability-hides-bash` | `probe()` 探不到 shell → `bash` 从 schema 消失（不是调了才报错）；与 G2 同一机制的不同 seam |
| H4 | `73-net-proxy-allowlist` | 网络独立一面：default deny + npm registry 白名单；首次新域名 → 一次 approve（**非静默**） |
| H5 | `74-fail-closed-doctor` | 依赖缺失 → 装配失败 + 列出"没过哪道门"；`agentic doctor` 打印 SandboxReport 与**实际生效等级** |
| H6 | `75-sandbox-derives-fs` | ★ 换 container 后端 → `read_file` 与 `bash` **一起**进容器（不会出现"容器里 cat 空、工具却读到内容"的语义分裂） |
| H7 | `76-escalation-request` | 被拦 → 违规信息回传模型 → 走 `approve` Submission 通道请求升权（升权词汇封闭，非扩权不打扰人） |

> H2 与 H6 是这组里最值得先看的两个：它们共同证明"沙箱不是一个工具的实现细节，而是**整个执行世界的边界**"——这也是我们把 `ctx.fs`/`ctx.subprocess` 默认 provider 交给 sandbox 派生的原因（SANDBOX.md §4.1）。

---

## Part 4 留给你的四个判断

1. **`@agentic/workflow` 与 `@agentic/teams` 是否独立成包**：独立则 surface 清晰（推荐），代价是多两个发布单元与版本轴。**建议独立。**
2. **Code-as-action（通道 C）是否一期做**：它对 Coding Agent 价值极高，但要求 `plugin-sandbox` 先落地。**沙箱已提到一期（SANDBOX.md §7），所以本条从"二期做"改成"一期做有条件版本"**：先上 `run_script`（仅 `native`/`container`），但不做 MITM、不做语言运行时隔离。
3. **示例的运行形态**：纯代码示例（`bun run`）vs 每例配一个 `.env.example` + 一个 mock provider（可离线跑、可当回归测试）。**建议后者**——尤其 B3 的假 provider，它同时是限流逻辑的唯一可靠测试资产。
4. **内置工具的命名与是否允许用户从面上摘除**：改名（`fs_read` → `read_file`）与 `task` 是否算第 8 个，两条都会写进对外契约且以后改不动。详见 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §7，那里还有 skills 规模墙与 `toolFromFunction` 是否公开两个待拍板点。
