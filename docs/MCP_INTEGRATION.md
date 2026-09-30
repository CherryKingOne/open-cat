# MCP 支持方案（设计与选型）

> 状态：**规划稿，待评审**。定位：MCP 与本地 tool 是**并列的两类能力来源**，都通过同一个 `ctx.tools` seam 汇入模型可见工具集；MCP 本身也是一个插件（`plugin-mcp`），不是特例路径。
> 前置阅读：[ARCHITECTURE.md](./ARCHITECTURE.md) §3 Seam、§4 事件契约。

---

## 1. 核心决策：MCP 在 Harness 里是什么

一句话：**MCP 是 `ctx.mcp` 这个 seam 的一个 provider，它的产物是"注册进 `ctx.tools` 的工具"，它的风险由 `ctx.policy` 的 waterfall 监听者统一接管。**

这样定的三个好处：
1. 模型侧不区分本地 tool 与 MCP tool —— 都是 ToolDefinition + schema，prompt 装配只认 `ctx.tools`。
2. MCP 可被整体卸载：`plugin-mcp` 的注册全是 `ctx.effect`，`agent.dispose()` 时子进程、监听器、工具注册一并逆序回收（不留僵尸进程）。
3. 换协议实现不影响上层：官方 SDK 只是 provider 的内部细节，将来换成自研 JSON-RPC 客户端，`ctx.tools` 与循环代码一行不动。

**对齐 dsh 的分类**：MCP 属于「集成与连接（integration）」能力族，而不是「工具（tools）」族——它是**连接外部系统的通道**，工具是它的产物之一（另外还有 resources 与 prompts）。

---

## 2. 协议与传输：支持哪几种，怎么选

| 传输 | 一期 | 用途 | 我们的启动/连接形态 |
|---|---|---|---|
| **stdio** | ✅ 主力 | 本地进程型 server（生态里 90%） | `Bun.spawn` 直接拉起，不依赖 Node |
| **Streamable HTTP** | ✅ | 远程/常驻 server、官方推荐 | POST + `Mcp-Session-Id`，可选 SSE 流 |
| **SSE（legacy）** | ⚠️ 兼容层 | 老 server | 通过 `mcp-remote` 代理，不原生实现 |
| WebSocket | ❌ | 规范外 | 不做 |

关于 `npx` 的定位（你问的那点）：**`npx` 不是管理机制，只是一种"启动配方"**。生态惯例是 `{ command: "npx", args: ["-y", "some-mcp-pkg"] }`，但真实执行链是"每次冷启动去 registry 拉包"，慢且不确定。我们的处理：

- **默认用 `bunx`**（同一语义，缓存命中时快一个数量级）；配置里写 `runtime: "npx" | "bunx" | "node" | "python" | "uv" | "command"`，由我们生成正确的 argv。
- **强制版本固定**：`pkg@<exact>` 或 lock 解析结果；`latest` / `main` 在生产 profile 里直接拒绝（dsh 社区文档同样强调这点："今天能跑，作者一更新明天崩"）。
- **预热命令** `agentic mcp install`：提前把包拉进缓存，避免首次工具调用卡在下载超时上（这是 MCP 接入最常见的"玄学超时"）。
- **Windows 适配**（我们的运行环境）：`npx`/`bunx` 实际是 `.cmd`，`Bun.spawn` 不能直接 exec → 统一走 `{ command: process.execPath, args: [bunxShim, ...] }` 或显式 `shell: true`，并在文档里给出绝对路径 fallback（很多客户端因为 `npx` 不在 PATH 而连不上）。

---

## 3. 协议实现选型

**建议：一期包一层官方 `@modelcontextprotocol/sdk` 的 Client，但严格封在 provider 内部。**

理由：MCP 的版本协商（`protocolVersion`）、能力声明（capabilities）、session 头、`elicitation`/`sampling` 这些细节繁多，自己实现一期不值得；但**必须**藏在 `ctx.mcp` 后面，因为：
- 官方 SDK 面向 Node，`stdio` transport 内部用 `cross-spawn` + Node streams，在 Bun 下有兼容成本，将来可能要自研（我们已经有 JSON-RPC over stdio 的踩坑清单：stdout 污染、换行黏包、请求 id 关联、进程优雅关闭）。
- 包一层之后，"实现可换"正是 seam 的本意。

```
packages/plugins/mcp/src/
  index.ts               plugin-mcp 入口（apply(ctx)）
  transport/
    stdio.ts             Bun.spawn 适配 + 换行分帧 + 进程树回收
    http.ts              Streamable HTTP + session 头
    remote.ts            mcp-remote 代理（legacy SSE）
  provider.ts            ctx.mcp 的默认 provider（内部依赖官方 SDK）
  naming.ts              mcp__server__tool 映射与冲突处理
  bridge.ts              MCP ToolDefinition ⇄ 我们的 ToolDefinition
  discovery.ts           tools/list 结果缓存 + 清单哈希
  pool.ts                连接池 / 懒连接 / 重连 / 熔断
  config.ts              mcp.json 解析与 schema 校验
```

---

## 4. 配置形态（管理方式）

默认文件 `mcp.json`（项目根或 `DSH_HOME` 式的应用目录），**顶层 key 兼容生态惯例 `mcpServers`**，这样用户能直接把 Cursor / Claude Desktop 的配置粘过来：

```jsonc
{
  "mcpServers": {
    "filesystem": {
      "runtime": "bunx",
      "package": "@modelcontextprotocol/server-filesystem@2026.1.0",  // 强制精确版本
      "args": ["./workspace"],
      "env": { "ROOT": "${WORKSPACE_ROOT}" },        // 只允许 ${VAR} 引用，不写明文密钥
      "enabled": true,
      "capabilities": { "tools": true, "resources": false, "prompts": false },
      "includeTools": ["read_file", "search_files"], // 白名单优先
      "excludeTools": [],
      "approval": "auto",                            // auto | ask | deny-list
      "lazy": true,                                  // 首次调用才连接
      "timeout": { "connect": 15000, "call": 60000 },
      "maxResultBytes": 65536                        // 超出则卸载到 session log 引用
    },
    "company-api": {
      "url": "https://mcp.example.com/mcp",          // Streamable HTTP
      "headers": { "Authorization": "Bearer ${TOKEN}" },
      "approval": "ask"
    }
  },
  "trust": {
    "pinLockfile": "mcp.lock.json",   // 记录解析版本 + tools/list 清单哈希
    "alertOnToolListChange": true     // 防 tool poisoning / 描述注入
  }
}
```

配套命令（CLI 侧，一期只做这三条）：

| 命令 | 作用 |
|---|---|
| `agentic mcp install` | 预热包缓存；写/更新 `mcp.lock.json` |
| `agentic mcp doctor [server]` | 逐个连接 → `initialize` → `tools/list`，打印发现的工具、协议版本、耗时、失败原因 |
| `agentic mcp list` | 输出当前 profile 下实际注册进 `ctx.tools` 的全量工具（本地 + MCP，含命名映射表） |

`doctor` 是**必须的**，不是可选项：MCP 排障的 90% 成本在"到底连上了没、发现了几个工具、名字映射成什么了"。dsh 用 `--dump-config` 解决同类问题，我们用 `mcp doctor` + `agent.dumpConfig()` 对偶解决。

Profile 集成：`mcp` server 列表也是可被 patch 覆写的配置行，因此 `sdk-minimal` 可以直接不含 MCP bundle（体积与冷启动考虑）。

---

## 5. 工具命名与冲突（细节决定能不能落地）

模型 provider 对 tool name 通常只允许 `[a-zA-Z0-9_-]`，所以**用双下划线，不用冒号**：

```
mcp__<serverName>__<toolName>
例：mcp__filesystem__read_file
```

规则：
1. `serverName` / `toolName` 先做字符集清洗（非法字符 → `_`），清洗后**必须可逆**：注册表保留 `原名 ⇄ 映射名` 双向表，调用时还原成 server 侧真实名字。
2. 清洗后仍冲突 → 追加短哈希后缀 `_<6>`，并在 `mcp list` 中显式打印冲突项。
3. 单 server 工具数上限（默认 64）；超出则强制走"渐进披露"模式（§7）。
4. 名字里必须让模型**看见**这是外部能力（前缀保留），否则无法做正确的调用决策，也不利于安全审计。

---

## 6. MCP 三类能力的映射（不要只接 tools）

| MCP 能力 | 映射到我们的哪里 | 一期 |
|---|---|---|
| **Tools** | 注册进 `ctx.tools`，schema 原样透传 + `annotations` 转成风险标记 | ✅ |
| **Resources** | ① 生成 `mcp__<s>__read_resource` 单工具；② 支持在 prompt 里以 `@server:uri` 引用，装配阶段展开成消息内容 | ①✅ ②二期 |
| **Prompts** | 注册进 `ctx.commands`（人工 slash 命令，**不过模型 turn**） | ⚠️ 视 commands 是否一期 |
| **Sampling**（server 反向请求 `createMessage`） | 路由回 `ctx.llm`，且**强制过 `ctx.policy`**：默认 `ask` | 二期（安全敏感） |
| **Elicitation**（server 请求补充输入） | 转成审批事件，走 `tools/pre-execute` waterfall 的 `ask` 分支 | 二期 |
| **Roots**（server 请求可见目录边界） | 从 `ctx.fs` 的 scope 导出，作为 workspace 边界告知 server | ⚠️ 随 fs |
| **Progress** | 转成 live 事件 `mcp/progress`，可被 UI/日志消费 | ✅ |
| **Cancellation** | `AbortController` 贯穿到 transport；进程型 server 走优雅关闭 + 超时强杀 | ✅ |

`annotations` 是 MCP 给的现成风险信号（`readOnlyHint` / `destructiveHint` / `openWorldHint` / `idempotentHint`），直接喂给默认 policy：

| annotation | 默认决策 |
|---|---|
| `readOnlyHint` | allow |
| 无标注（未知） | allow + 记 log（一期不激进） |
| `destructiveHint` | **ask** |
| `openWorldHint`（会触达外部网络/真实世界） | **ask** |

`ask` 一期落地为：抛出 `policy/decision` 事件，无人监听时按 `fail: 'deny'` 处理（安全默认）。人工审批 UI 属于宿主职责，SDK 只提供注入决策的监听者接口。

---

## 7. 上下文成本治理（接 MCP 后必然踩的坑）

接三个 MCP server 就可能往 prompt 里塞 200 个工具、几万 token 的 schema。这是"接了 MCP 之后 Agent 变笨"的真实原因，属于 harness 层职责，必须在设计里给出答案：

1. **清单缓存**：`tools/list` 结果作为投影存进 session 层，不重复拉取；哈希进 lock 文件用于变更告警。
2. **渐进披露（toolbox 模式）**：当某 server 工具数 > 阈值，或全量工具 schema 估算 token > 预算的 X%，该 server 只暴露两个元工具：
   - `mcp__<s>__search_tools(query)` → 返回候选工具及 schema
   - `mcp__<s>__call_tool(name, args)` → 实际执行
   命中后把该工具"提升"为本轮临时可见工具。**默认开启，可关。**
3. **输出卸载（offload）**：工具结果超过 `maxResultBytes` → 全文写入 session log 作为 durable artifact，模型只看到截断预览 + `ref`；需要全文时用 `read_artifact(ref)` 取。对齐 dsh 的"把工具输出卸载到磁盘"。
4. **工具路由（Tool Router）**：`tools/pre-execute` waterfall 上允许插件做"按意图筛选/重排工具集"，二期。

---

## 8. 生命周期与健壮性（stdio 型是重灾区）

```
配置解析 → 懒连接判定 ─┬─ lazy:false → 启动即 initialize
                      └─ lazy:true  → 首次 tools/list 或 call 时连接
   initialize（协议版本协商 + capabilities 交换）
   tools/list → 命名映射 → 注册进 ctx.tools（注册即 ctx.effect）
   call → waterfall(pre-execute) → execute → waterfall(post-execute) → 结果
   断连/进程退出 → 指数退避重连（上限 N 次）→ 熔断标记 unhealthy → 工具临时下架并在 schema 中标注
   dispose → 逆序回收：取消在途请求 → 关闭 transport → 杀进程树（Windows 用 taskkill /T）
```

必须写进实现清单的踩坑点（都是 MCP 集成高频故障）：
- **stdout 污染**：server 往 stdout 打日志会直接打碎 JSON-RPC 帧 → 我们侧强制只解析换行分隔的合法 JSON，非帧内容重定向到 stderr 并记 log；文档里告知 server 作者。
- **黏包/半包**：按行分帧 + 缓冲，禁止"一次 read 当一个消息"。
- **请求 id 关联与超时**：每个请求登记 id → pending，超时也要清理，否则内存泄漏。
- **进程树回收**：Node/npx 会派生子进程，只杀父进程会留下持有 stdout 管道的孤儿 → 平台化的进程树终止。
- **首次 npx 冷启动超时**：靠 §4 的 `install` 预热解决，默认 connect timeout 15s 不足以覆盖下载。
- **优雅关闭与 in-flight**：`UNLOADING` 状态必须先等在途请求结束或取消，再杀进程（对应 Cordis 的 withdrawal 语义）。

---

## 9. 安全（对外 SDK 必须写进 README 的部分）

1. **版本固定 + 清单哈希告警**：防上游篡改与 **tool poisoning**（server 更新后偷偷加入危险工具或改描述）。哈希变化默认拒绝注册，需显式确认。
2. **描述即注入面**：MCP server 的 tool description 与返回内容都是不可信输入，会进 prompt。默认策略：结果里的高亮指令不做特殊处理，但在 `tools/post-execute` 上提供可选的敏感模式（外部内容包裹标记），二期。
3. **密钥不入配置文件明文**：`env` 只允许 `${VAR}` 形式，从凭据服务读取；`dumpConfig()` 输出必须脱敏。
4. **执行世界一致性**（沿用 dsh 的思路）：stdio 型 server 是本地任意代码执行。它必须与 `ctx.sandbox` / `ctx.fs` scope 共享同一个执行世界——否则你在循环里做了沙箱，MCP server 却能在沙箱外读写。这条要在实现时作为硬性约束。
5. **默认 profile 不自动连接任何外部 server**；`sdk-minimal` 不含 MCP。
6. **审计**：所有 MCP 调用落 durable `tool/call` / `tool/result` 事件，包含 server、映射名、原始名、policy 决策结果。

---

## 10. 一期 / 二期切分（建议）

**一期（MVP）**：stdio + Streamable HTTP；bunx/npx 启动配方与版本固定；`tools` 全量映射与命名；`doctor` / `install` / `list` 三条命令；annotation→policy 默认；输出卸载；进程树回收；懒连接与重连。

**二期**：resources 的 `@` 引用展开、prompts→commands、sampling、elicitation、progress 可视化、工具路由、远程沙箱共享执行世界、把我们的 Agent 反向导出为 MCP server（`agentic serve mcp`）。

---

## 11. 需要你拍板的问题

1. 一期是否接受"包一层官方 `@modelcontextprotocol/sdk`"？（备选：自研最小 JSON-RPC over stdio 客户端，约多 3~5 天工作量，换来零 Node 依赖与完全可控）
2. 配置格式：**兼容生态的 `mcpServers`**（利于用户粘贴迁移）还是自创 `@agentic` 专属 schema（利于表达 profile/patch）？文档里现在是"兼容 + 扩展字段"。
3. 渐进披露（toolbox 模式）默认**开**还是**关**？默认开更省 token，但会改变模型行为、排查更绕。**建议默认开 + 阈值可配。**
4. 版本固定强制到什么程度：`latest` 是报错（安全）还是警告（易用）？**建议：生产 profile 报错，dev profile 警告。**
5. `approval: ask` 一期如何落地：SDK 只提供事件与注入接口，还是内置一个 CLI 交互式确认？**建议前者 + CLI 里给一个默认监听者。**
6. 是否要支持 **profile 级不同的 MCP 集合**（例如 `sdk-minimal` 不带、`cli` 全带）？这与 ARCHITECTURE §6 的分层覆写直接相关。
