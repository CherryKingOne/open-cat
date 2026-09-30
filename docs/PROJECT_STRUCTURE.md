# 项目目录结构规划（agentic · Harness 版）

> 状态：**v2.4 已定稿，进入实施**（v1 扁平 SDK 包结构；v2 按 [ARCHITECTURE.md](./ARCHITECTURE.md) 的微内核 + 插件重构；v2.1 回填 [HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md) 的 12 项借鉴；v2.2 回填 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md)；v2.3 回填 [SANDBOX.md](./SANDBOX.md)；**v2.4：§7 的 11 个拍板点全部按建议值定稿，见 [ARCHITECTURE.md](./ARCHITECTURE.md) §9**）。
> 定位：基于 **Bun** 手动自定义开发的 **Agent Harness**，Agent 创建入口唯一为 **`createReactAgent()`**，面向对外开放与 SDK 生成。
> 设计参照：DeepSeek Harness（dsh）/ Cordis —— "一切皆插件"，没有需要打补丁的特权内核。
> 早期为纯设计稿（只出文档不写代码）；**v2.4 起本文与 [ARCHITECTURE.md](./ARCHITECTURE.md) 同为工程约束，代码按 §8 顺序从此衍生**。实现与本文不一致时，以代码 + CI 断言为准，并回头修本文。

---

## 1. 设计原则

| 原则 | 说明 | 来源 |
|---|---|---|
| **Agent = Model + Harness** | 模型提供原始智能，harness 提供可信赖/可观测/可控制的全部工程支撑 | dsh 官方定义 |
| **一切皆插件** | 模型适配、工具注册表、会话日志、**Agent 循环本身**都是插件，都可从配置替换 | dsh 架构文档 |
| **没有特权内核** | 扩展方式是"在旁边挂一个插件"，不是 fork 改源码 | dsh 架构文档 |
| **Seam 三角色纪律** | 一项能力 = Definition + Provider + Consumer，缺一不算 seam | dsh `capability seams` |
| **注册即可撤销** | 一切上下文变更归结到 `ctx.effect`，卸载逆序回滚 → 热重载、测试隔离、无泄漏 | Cordis reversible effects |
| **依赖即装配顺序** | `inject` 声明服务依赖，运行时推导启停顺序，无硬编码启动序 | Cordis reactive coeffects |
| **策略走事件，不改循环** | 拦截/改写/拒绝用 `waterfall` 事件监听者，而非固定中间件数组 | dsh `agent/pre-step`、`tools/*` |
| **模型可见 = 已记录** | 模型能看到的输入必须先是一条 session event；历史由 log 唯一投影 | dsh session log 不变量 |
| **分层组装** | Profile（配方）+ Bundle（分发单元）+ Patch（逐层覆写）+ 可 dump | dsh profile/bundle |
| **SDK-First** | 对外契约是 `import` 与类型；配置组装是内部机制，对外只暴露 `patches` | 本项目与 dsh 的有意偏离 |
| **Bun 原生** | `bun install/build/test`；不引入 vite/webpack/ts-node | — |

---

## 2. 目录结构树

```
agentic/
├── package.json                      # 根：private + workspaces + 统一 scripts
├── bun.lock
├── bunfig.toml                       # Bun 配置（test preload、env）
├── tsconfig.json                     # 严格模式 + project references
├── .npmrc                            # registry / access
├── .gitignore
├── mcp.schema.json                   # mcp.json 的 JSON Schema（对外契约之一）
│
├── .github/workflows/
│   ├── ci.yml                        # lint / typecheck / bun test / 结构校验三兄弟
│   └── release.yml                   # 版本一致性 + build + publish（tag 触发）
│
├── docs/
│   ├── PROJECT_STRUCTURE.md          # 本文档
│   ├── ARCHITECTURE.md               # ★ harness 内核、seam、事件、turn/step 设计
│   ├── MCP_INTEGRATION.md            # ★ MCP 接入、传输、管理、安全
│   ├── LOOP_ENGINEERING.md           # ★ 教程：循环设计、终止条件、限流退避、Dots 对照
│   ├── CONTEXT_BUDGET_MEMORY.md      # ★ 教程：五层阈值、缓存前缀、Context 关联、Memory
│   ├── REPO_INTELLIGENCE.md          # ★ 教程：代码理解层（不再只读 README）
│   ├── SDK_SURFACE.md                # ★ Tools 定义、五种能力通道（含 Skills）、三 surface 导入路径、案例规划
│   ├── HARNESS_CASE_STUDIES.md        # ★ dsh / pi / Codex / 回退系对比与我们的取舍裁决
│   ├── BUILTIN_TOOLS.md               # ★ 内置原子工具、Skills、工具格式契约（Deep Agents 调研）
│   ├── SANDBOX.md                     # ★ 沙箱：各家怎么实现 + 我们的沙箱 SDK（Codex / Claude Code / letta / dsh / 云端厂商）
│   ├── SEAM_CATALOG.md               # seam 清单（Definition/Provider/Consumer 三列）
│   ├── EVENT_MAP.md                  # 事件全集 + producer/consumer + durable/live 标记
│   ├── SDK_GUIDE.md                  # 对外使用指南
│   └── FORKING_PLUGINS.md            # 如何替换某个能力而不 fork（插件开发指南）
│
├── packages/
│   │
│   ├── kernel/                       # @agentic/kernel —— 微内核，只有组合语义
│   │   ├── package.json              # publishConfig.access: public
│   │   ├── src/
│   │   │   ├── index.ts              # 导出：createContext / defineSeam / Service / Plugin 类型
│   │   │   ├── context.ts            # ctx：fork / provide / inject / effect / on / emit
│   │   │   ├── effect.ts             # ★ 唯一副作用原语（其余注册全是它的特例）
│   │   │   ├── service.ts            # ServiceDefinition + 注册表 + 版本
│   │   │   ├── inject.ts             # 响应式共效：依赖满足才激活，提供方变化自动重启
│   │   │   ├── event/
│   │   │   │   ├── bus.ts            # 分发实现
│   │   │   │   └── dispatch.ts       # emit / waterfall / parallel / serial
│   │   │   ├── fiber.ts              # PENDING→LOADING→ACTIVE→UNLOADING→DISPOSED + FAILED
│   │   │   ├── plugin.ts             # Plugin 契约与 mount/递归卸载
│   │   │   ├── seam.ts               # defineSeam：强制三角色齐全
│   │   │   └── disposable.ts
│   │   └── tests/                    # effect 逆序回滚、依赖重启、层级隔离
│   │
│   ├── types/                        # @agentic/types —— 纯类型，零运行时依赖
│   │   └── src/
│   │       ├── index.ts
│   │       ├── seam.ts               # 各 seam 的接口（LlmProvider / ToolRegistry / AgentLoopDriver…）
│   │       ├── session-event.ts      # ★ durable 事件联合类型（真相源契约）
│   │       ├── events.ts             # EventMap：事件名 → payload → 分发模式
│   │       ├── tool.ts               # ToolDefinition / annotations（readOnly/destructive…）
│   │       └── content.ts            # ContentPart / Usage / FinishReason
│   │
│   ├── core/                         # @agentic/core —— SDK 门面（刻意极薄）
│   │   ├── src/
│   │   │   ├── index.ts              # 根导出：createReactAgent + defineProfile/defineBundle + 类型
│   │   │   ├── react-agent.ts        # createReactAgent：解析 profile → 建 ctx → 装 fiber 树 → 返回句柄
│   │   │   ├── agent-handle.ts       # run / stream / inject / fork / resume / dispose / dumpConfig
│   │   │   ├── profile.ts            # defineProfile：具名配方 + patch 分层叠加
│   │   │   ├── bundle.ts             # defineBundle：插件 + 配置行的可分发单元
│   │   │   ├── patch.ts              # 按 id 整行替换 / 插入（顺序：bundle 序 → profile → 用户 → --patch）
│   │   │   ├── use-sugar.ts          # agent.use() → 内部转 waterfall 监听者（语法糖，非内核概念）
│   │   │   └── internal/             # ★ 不对外导出
│   │   └── tests/
│   │
│   ├── plugins/                      # ★ 能力提供者，插件之间零依赖，只通过 ctx key 协作
│   │   ├── agent-loop-react/         # @agentic/plugin-agent-loop-react —— ReAct 循环（ctx.agentLoop 默认 provider）
│   │   │   └── src/{index,loop,step,turn,inbox,cancellation,retry}.ts   # inbox = ctx.inbox 默认 provider（插话分类：append\|newTask\|interrupt）
│   │   ├── llm/                      # @agentic/plugin-llm —— ctx.llm seam + 消息/流词汇
│   │   │   └── src/{index,capabilities,stream-vocab}.ts
│   │   ├── tools/                      # @agentic/plugin-tools —— ctx.tools：工具作用域注册表 + 守卫执行管线 + **能力探测隐面**
│   │   │   └── src/{index,registry,scope,define-tool,pipeline,capabilities,result,decorators/{guards,retry,cache}}.ts   # decorators = T2 的“像继承的体验”（`@agentic/plugin-tools/operators`）
│   │   ├── session/                    # @agentic/plugin-session —— ctx.sessions：append-only **树** log（parentId）+ deriveMessages
│   │   │   └── src/{index,log,derive,projection,persistence,jsonl-v0}.ts
│   │   ├── checkpoint/                 # ★ @agentic/plugin-checkpoint —— ctx.checkpoints：消息状态与世界状态一起原子回滚（消息投影 + 影子 git）
│   │   │   └── src/{index,snapshot,shadow-git,rollback,compare,prune}.ts   # 见 HARNESS_CASE_STUDIES §2.4
│   │   ├── system-prompt/              # @agentic/plugin-system-prompt —— 分段装配 + tool schema 汇入
│   │   ├── context/                    # @agentic/plugin-context —— 压缩/裁剪 + **结果驱逐到 /artifacts/**（挂 agent/request waterfall）
│   │   ├── memory/                     # @agentic/plugin-memory —— ctx.memory + memory_* 工具
│   │   ├── fs/                         # ★ @agentic/plugin-fs —— ctx.fs：七方法 Backend + capabilities + routes（一期必做）
│   │   │   └── src/{index,backend,virtual-mode,permissions,routes,tools/{ls,read-file,write-file,edit-file,glob,grep}}.ts
│   │   ├── subprocess/                 # ★ @agentic/plugin-subprocess —— ctx.subprocess（Bun.spawn）+ tools/bash.ts（一期必做）
│   │   ├── skills/                     # ★ @agentic/plugin-skills —— ctx.skills：Agent Skills 规范的发现/分级/事件
│   │   │   └── src/{index,discover,frontmatter,sources,prefilter,trust,events}.ts   # 见 BUILTIN_TOOLS §3
│   │   ├── planning/                   # @agentic/plugin-planning —— ctx.planning：write_todos（**opt-in**，不入 core 面）
│   │   ├── sandbox/                    # @agentic/plugin-sandbox —— ctx.sandbox：与 fs/subprocess 共享执行世界〔接口先行〕
│   │   ├── mcp/                        # ★ @agentic/plugin-mcp —— 见 MCP_INTEGRATION.md
│   │   │   └── src/
│   │   │       ├── index.ts            # apply(ctx)：连接池 → tools/list → 命名映射 → 注册进 ctx.tools
│   │   │       ├── config.ts           # mcp.json 解析与校验（兼容 mcpServers）
│   │   │       ├── provider.ts         # ctx.mcp 默认 provider（内部包官方 SDK，可整体替换）
│   │   │       ├── naming.ts           # mcp__server__tool 清洗/可逆/冲突
│   │   │       ├── bridge.ts           # MCP tool ⇄ ToolDefinition（annotations → policy 标记）
│   │   │       ├── discovery.ts        # tools/list 缓存 + 清单哈希 + 变更告警
│   │   │       ├── pool.ts             # 懒连接 / 保活 / 退避重连 / 熔断下架
│   │   │       ├── disclosure.ts       # 渐进披露：search_tools / call_tool 元工具
│   │   │       └── transport/
│   │   │           ├── stdio.ts        # Bun.spawn + 换行分帧 + 进程树回收（Windows taskkill /T）
│   │   │           ├── http.ts         # Streamable HTTP + Mcp-Session-Id
│   │   │           └── remote.ts       # mcp-remote 代理（legacy SSE）
│   │   ├── policy/                     # ★ @agentic/plugin-policy —— ctx.policy + tools/pre-execute 的 ask/deny + **sensitive-file-rules.ts**（kimi 式硬过滤：.env / id_* / credentials / .pem|.key|.old）
│   │   ├── sandbox/                    # ★ @agentic/plugin-sandbox —— ctx.sandbox 装配：**派生** ctx.fs 与 ctx.subprocess（一个换、两个跟着换）+ `sandbox/probe` durable 事件 + **fail-closed 默认**
│   │   ├── workflow/                   # ★ @agentic/plugin-workflow —— 在 ctx.agentLoop 之上的编排 provider（不新建运行时）
│   │   │   └── src/{index,sequence,parallel,branch,loop,nodes,operators/{retry,human,checkpoint,compensate}}.ts
│   │   ├── teams/                      # ★ @agentic/plugin-teams —— 多 Agent：roster + 任务板 + mailbox（opt-in seam）
│   │   │   └── src/{index,supervisor,pipeline,handoff,task-board,mailbox,blackboard,budget-shard,worktree,tools/task}.ts   # task = 内置子 agent 工具；worktree = 并行隔离单位
│   │   ├── repo-intel/                 # ★ @agentic/plugin-repo-intel —— ctx.repoIntel：代码知识图谱 + repo_overview
│   │   │   └── src/{index,overview/{structure,conventions,intent-reality,gaps},graph,query,history,impact}.ts
│   │   ├── observability/              # @agentic/plugin-observability —— telemetry/* emit + trace
│   │   └── commands/                   # @agentic/plugin-commands —— ctx.commands：人工命令不过模型 turn
│   │
│   ├── providers/                      # llm seam 的外部实现（与 plugins 分层：这里只放厂商适配）
│   │   ├── openai/                     # @agentic/provider-openai（同时覆盖 OpenAI 兼容端点）
│   │   ├── anthropic/                  # @agentic/provider-anthropic
│   │   └── ollama/                     # @agentic/provider-ollama
│   │
│   ├── sandbox/                      # ★ @agentic/sandbox（一期）—— 沙箱契约 + native 后端，**零厂商依赖**
│   │   ├── src/{index,backend,policy,capabilities,probe,net-proxy}.ts   # SandboxBackend / defineSandboxPolicy（策略即数据，可进 dumpConfig）/ 本地 HTTP CONNECT 代理
│   │   ├── src/native/{seatbelt,bwrap,windows,argv}.ts   # SBPL 模板参数化路径 / bwrap 挂载序（--tmpfs 遮蔽＝缺席，--die-with-parent，carve-out 不得为 deny 祖先）/ 一期引导 WSL2 或 container / argv 优先不拼 shell
│   │   ├── src/policies/*.sbpl.tpl   # 2×3 矩阵：permissive|restrictive × open|closed|proxied（形状抄 qwen-code，模板可单测快照）
│   │   └── src/testing/fake-sandbox.ts  # ★ CI 用假后端（probe 报告可注入，不依赖真 OS）
│   ├── sandbox-container/            # @agentic/sandbox-container（一期建议）—— docker/podman；会话级复用；默认 --network none --memory --cpus --cap-drop ALL
│   ├── sandbox-remote/               # @agentic/sandbox-remote（**二期**）—— e2b（互操作轴）/ daytona / modal adapter + **world 级 snapshot/hydrate**
│   ├── sandbox-iso/                  # @agentic/sandbox-iso（**二期**）—— B 类 CoW 工作区视图：APFS clonefile / overlayfs / btrfs / reflink / ProjFS / worktree 兜底
│   │
│   ├── bundles/                        # ★ 可分发组合包（bundle = 插件集 + 配置行）
│   │   ├── standard/                   # @agentic/bundle-standard —— session+llm+tools+loop+prompt+policy
│   │   ├── mcp/                        # @agentic/bundle-mcp —— plugin-mcp + 默认 policy 规则 + 命名配置
│   │   └── minimal/                    # @agentic/bundle-minimal —— 自带完整显式树，不叠 standard
│   │
│   └── cli/                            # @agentic/cli（对外可执行入口）
│       └── src/
│           ├── index.ts                # profile 启动器：agentic --profile <name>
│           ├── dump-config.ts          # ★ 打印本机实际组装的插件树与每行配置（可排障的关键）
│           ├── inspect.ts              # 列出 ctx 上的服务、seam 提供方、fiber 状态
│           ├── repl.ts                 # 交互式验证 ReactAgent 循环
│           └── mcp/                    # mcp install / doctor / list
│
├── profiles/                         # 内置 profile 模板（配方文件，不是代码）
│   ├── sdk/            # 默认：bundle-standard + bundle-mcp + policy
│   ├── sdk-minimal/    # 单 bundle 全显式树，不含 MCP，冷启动最快
│   ├── core8/          # ★ 零配置默认面：ls/read_file/write_file/edit_file/glob/grep/bash + task（=8，受 check-surface 断言）
│   ├── coding/         # ★ core8 + skills + planning + repo-intel（预算另立 ≤ 14）
│   ├── scout/          # ★ 只读模式：无任何写工具/无对外发消息；笼子＝read-only，`bash` 仍在面上但只能跑白名单只读命令
│   ├── cli/            # sdk + repl + commands
│   └── headless/       # sdk + one-shot runner，不起服务，供 CI/脚本
│
├── templates/                        # ★ 用户侧约定目录的模板（对外即“约定优于配置”的面）
│   └── dot-agentic/                  # 拷到项目里就是 .agentic/：零代码扩展的三个入口
│       ├── tools/example.ts          # 进程内自定义工具（defineTool 默认导出；默认不自动加载，需 trust）
│       ├── skills/example-skill/SKILL.md  # 技能：frontmatter(name/description) + 正文(< 5000 token) + scripts|references
│       ├── commands/precommit.md     # 人工斜杠命令（不过模型 turn）
│       ├── sandbox.json              # ★ 沙箱策略（mode/filesystem/network/limits/escalation）——与 mcp.json 同级的对外约定面
│       ├── sandbox.Dockerfile        # 项目自定义容器镜像（可选；FROM agentic-sandbox，形状抄 .gemini/sandbox.Dockerfile）
│       ├── mcp.json                  # MCP 声明（见 MCP_INTEGRATION）
│       └── AGENTS.md                 # 项目知识与约定（进 git、可 review）
│
├── apps/                             # ★ private，不发布
│   └── playground/src/main.ts        # 手工验证：起一个带 MCP 的 agent，跑通 turn/step
│
├── examples/                         # ★ 第二份文档。每例配 .env.example + mock provider（可离线跑 + 当回归资产）
│   ├── A-agent/                      # 基础
│   │   ├── 01-minimal-react-agent/   # 一次 turn 一个 step
│   │   ├── 02-define-tool/           # 五维 tool 定义（annotations/cost/describeError）
│   │   ├── 03-streaming/             # durable + live 混合投影
│   │   ├── 04-budget-degrade/        # 四维预算撞线 → 受控降级（partial + resumeToken）
│   │   ├── 05-resume-from-checkpoint/ # session log fork/resume + 幂等 journal
│   │   └── 06-replace-provider/      # patch 一行换模型，消费方自动重启
│   ├── B-control/                    # ★ 可控性（LOOP_ENGINEERING）
│   │   ├── 10-approval-waterfall/    # 破坏性动作 → awaiting-approval（可恢复状态）
│   │   ├── 11-loop-detection/        # 参数规范化哈希 + 进展指标 → replan
│   │   ├── 12-rate-limited-provider/ # ★ 假 provider：RPM→退避+jitter / TPM→降 token / 额度→停止
│   │   └── 13-scanner-always-on/     # 常驻形态：定时唤醒 + 自动去重 + Activity View
│   ├── C-teams/                      # ★ 多 Agent（@agentic/teams）
│   │   ├── 20-supervisor-worker/     # 上下文隔离（子 ctx 见父、父不见子）
│   │   ├── 21-pipeline-specialists/  # 交接契约：结构化工件而非自然语言接力
│   │   ├── 22-handoff-routing/       # 会话所有权转移 + 历史裁剪 + 预算随迁
│   │   └── 23-parallel-fanout-board/ # ★ 并发预算分片（否则子 agent 一起撞 TPM 墙）
│   ├── D-workflow/                   # ★ Workflow（@agentic/workflow）
│   │   ├── 30-sequence-basic/        # 每步一个 checkpoint
│   │   ├── 31-branch-and-loop/       # 条件边 + 受限循环
│   │   ├── 32-parallel-with-join/    # fail-fast vs best-effort + 并发闸
│   │   ├── 33-human-in-the-loop/     # 真持久化中断：进程可退出，数天后继续
│   │   ├── 34-agent-node-mixed/      # ★ 推荐生产形态：workflow 定骨架 + agentNode 填不确定节点
│   │   └── 35-saga-compensation/     # 补偿事务
│   ├── E-repo/                       # ★ 项目理解（REPO_INTELLIGENCE）
│   │   ├── 40-repo-overview/         # intentVsReality 标出 README 与现实的差异
│   │   └── 41-orient-execute-validate/ # 三阶段门禁 + 图优先纪律
│   ├── F-extensibility/              # ★ 一切皆插件的证据
│       ├── 50-write-a-plugin/        # 30 行替换一个能力，不 fork 代码
│       └── 51-effect-cleanup/        # dispose() 逆序回滚：子进程/定时器/监听器全回收
│   ├── G-tools-skills/               # ★ 内置工具与技能（BUILTIN_TOOLS）
│       ├── 60-zero-config-builtin/   # ★ 不传一个 tool，拿到 8 个原子工具就能改仓库
│       ├── 61-capability-hiding/     # 不支持 delete 的 backend → `delete` 从 schema 里消失
│       ├── 62-artifact-offload/      # 大结果 → /artifacts/ → 模型自己 read_file(offset,limit) 读回
│       ├── 63-skill-progressive/     # L1 目录 → L2 read_file → L3 资源；+ skills/activated 事件
│       ├── 64-custom-tool-in-process/ # .agentic/tools/*.ts（含 trust 开关）
│       ├── 65-wrap-a-backend/        # ★ withGuards / wrapFs：装饰而非继承类（T2 扩展点）
│       └── 66-edit-file-failure/     # 失败可行动：NOT_UNIQUE / NOT_FOUND 的回填文本
│   └── H-sandbox/                    # ★ 沙箱（SANDBOX）
│       ├── 70-native-workspace-write/ # ★ 一行声明笼子：`sandbox: 'native:workspace-write'`，bash 能写工作区但写不了宿主
│       ├── 71-deny-is-absence/       # deny ~/.ssh → 目录直接“缺席”（tmpfs / 不 bind），而不是 EACCES；read_file 与 cat 答案一致
│       ├── 72-capability-hides-bash/ # probe 探不到 shell → bash 从 schema 消失（不是调了才报错）
│       ├── 73-net-proxy-allowlist/   # default deny + npm registry 白名单；首次新域名 → 一次 approve（非静默）
│       ├── 74-fail-closed-doctor/    # 依赖缺失 → 装配失败 + 列出“没过哪道门”；`doctor` 打印 SandboxReport
│       ├── 75-sandbox-derives-fs/    # ★ 换 container 后端 → read_file 与 bash 一起进容器（不会出现语义分裂）
│       └── 76-escalation-request/    # 被拦 → 违规信息回传模型 → 走 approve 通道请求升权（封闭词汇、非扩权不打扰人）
│
├── scripts/                          # ★ 规范守门（这些校验脚本是 SDK 质量的地基，不能省）
│   ├── check-exports.ts              # exports 字段 ⇄ src/index.ts 导出面一致；internal 不外泄
│   ├── check-deps.ts                 # 依赖方向单向；plugins 之间零 import（只能走 ctx key）
│   ├── check-seams.ts                # 每个 seam 三角色齐全；事件必须声明分发模式
│   ├── check-grepability.ts          # ★ 会话文件必须可 grep/jq 直读；禁止模型可见内容只存于压缩/二进制表示
│   ├── check-surface.ts              # ★ profile 能力面断言：core profile 工具数 ≤ 8、system prompt token ≤ 1.2k
│   ├── check-tools.ts                # ★ 工具契约：命名正则/内置名白名单/strict schema/每个工具至少一条 ok:false 路径
│   ├── check-skills.ts               # ★ SKILL.md：frontmatter 合法/正文 <5000 token/路径必为“含技能目录的目录”
│   ├── check-sandbox.ts              # ★ 沙箱守门：策略覆盖算法三条/carve-out 不得为 deny 祖先/fail-closed 为默认/SBPL 与 bwrap 参数生成快照/敏感文件规则表
│   ├── verify-event-map.ts           # EVENT_MAP 与 types/events.ts 一致
│   └── release.ts                    # 版本一致性 + build + publish
│
└── third_party/licenses/             # 依赖许可证归档（商用开放前置合规）
```

---

## 3. 包规划

| 包 | 定位 | 发布 | 依赖 |
|---|---|---|---|
| `@agentic/kernel` | 微内核：ctx/service/event/effect/fiber/seam | ✅ | types |
| `@agentic/types` | 纯类型契约（seam 接口、SessionEvent、EventMap） | ✅ | 无运行时依赖 |
| `@agentic/core` | **SDK 门面**：`createReactAgent` + profile/bundle/patch | ✅ | kernel, types |
| `@agentic/workflow` | **Workflow surface**：序列/并行/分支/人工/补偿（建立在 agentLoop 之上的 provider，不新建运行时） | ✅ | kernel, types, plugin-workflow |
| `@agentic/teams` | **Team surface**：多 Agent 协作（任务板 / mailbox / 并发预算分片） | ✅ | kernel, types, plugin-teams |
| `@agentic/plugin-*` | 能力提供者，一个能力一个包 | ✅ | kernel, types |
| `@agentic/plugin-fs` / `-subprocess` | ★ 内置原子工具的**执行底座**（Backend 七方法 + capabilities + routes / Bun.spawn），一期必做 | ✅ | kernel, types |
| `@agentic/plugin-skills` | ★ ctx.skills：Agent Skills 规范的发现 / 分级 / 命中事件 | ✅ | plugin-fs, types |
| `@agentic/plugin-planning` | ctx.planning：`write_todos`（**opt-in**，不入 core 工具面） | ✅ | kernel, types |
| `@agentic/plugin-checkpoint` | ★ ctx.checkpoints：消息状态与世界状态**双栈一起**原子回滚（影子 git） | ✅ | kernel, types |
| `@agentic/sandbox` | ★ **沙箱契约 + native 后端**（Seatbelt/bwrap/probe/策略/假后端），零厂商依赖；可**单独 import 当执行环境用** | ✅ | types |
| `@agentic/plugin-sandbox` | ★ ctx.sandbox 装配：**派生 ctx.fs 与 ctx.subprocess** + `sandbox/probe` 事件 + fail-closed | ✅ | sandbox, kernel, types |
| `@agentic/sandbox-container` | docker/podman 后端（会话复用 + 资源上限）——一期建议 | ✅ | sandbox |
| `@agentic/sandbox-remote` / `-iso` | 二期：e2b/daytona/modal 适配 + world 级快照 / CoW 工作区视图 | ✅（二期） | sandbox |
| `@agentic/provider-*` | llm seam 的厂商实现 | ✅ | types, plugin-llm |
| `@agentic/bundle-*` | 插件 + 配置行的组合包 | ✅ | 对应 plugins |
| `@agentic/cli` | 启动器 + dump-config/inspect/mcp 子命令 | ✅ | core, bundles |

### 3.1 三个 surface + 横切扩展点的导入路径（对外契约，详见 [SDK_SURFACE.md](./SDK_SURFACE.md)）

```
① Agent    →  import { createReactAgent, defineProfile } from '@agentic/core'
② Workflow →  import { defineWorkflow, sequence, parallel, branch } from '@agentic/workflow'
              import { human, checkpoint, retry }                  from '@agentic/workflow/operators'
              import { agentNode }                                  from '@agentic/workflow/nodes'
③ Team     →  import { defineTeam, supervisor, pipeline, handoff } from '@agentic/teams'
              import { taskBoard, mailbox }                        from '@agentic/teams/coordination'

── 横切三个 surface（自定义工具与技能，见 BUILTIN_TOOLS.md §3.5 / §4）──
T1 定义   →  import { defineTool, ok, fail }              from '@agentic/plugin-tools'
T2 装饰   →  import { withGuards, withRetry, withCache }  from '@agentic/plugin-tools/operators'
T2 换后端 →  import type { FsBackend, ShellBackend }       from '@agentic/types'
T3 策略   →  import { defineProfile }                     from '@agentic/core'   // patch ctx.fs / ctx.subprocess 的 provider
约定加载 →  .agentic/tools/*.ts（默认关闭，需 trust） + .agentic/skills/<name>/SKILL.md（零代码）

── 沙箱（可给 agent 用，也可自己拿来当执行环境，见 SANDBOX.md §5）──
给 agent  →  import { nativeSandbox }  from '@agentic/sandbox/native'
                createReactAgent({ profile: 'core8', sandbox: nativeSandbox({ mode: 'workspace-write', deny: ['~/.ssh'] }) })
                或简写 sandbox: 'native:workspace-write'   // 字符串简写也要能在 dumpConfig() 里展开
单独使用 →  import { createSandbox }  from '@agentic/sandbox'          // create* → 必 dispose
                import { containerSandbox } from '@agentic/sandbox/container'
契约与策略 → import { defineSandboxPolicy } from '@agentic/sandbox'   + type SandboxBackend from '@agentic/sandbox'
测试     →  import { fakeSandbox }      from '@agentic/sandbox/testing'   // 不依赖真 OS 的 CI 资产
```

> **内置工具不需要 import**：`ls/read_file/write_file/edit_file/glob/grep/bash(+task)` 由 profile 装配时自动挂上，用户 `createReactAgent()` 后直接可用——这是第 1 问「基础工具无需手动定义」的落点，也是 `profiles/core8` 存在的全部理由。**沙箱同理：不声明就有默认档（`workspace-write`），声明只是换档。**

命名动词铁律（比目录结构更容易失守的地方）：`createX()` = 有生命周期、要 `dispose()`；`defineX()` = 无副作用、可序列化、可复用；`withX()` = 装饰；`on(event)` = 订阅并返回 `Disposable`。

**粒度决策**：一期采用 **plugin 独立包**（而非单包多子路径）。理由：`一切皆插件` 要求"换掉某个能力"是移除/替换一个提供方，独立包让这条路径最干净；代价是包数量多，用 `--filter` 批量构建 + 统一版本轴（changeset）缓解。**这是最需要你确认的一点，备选方案见 §7-1。**

> 命名待核：`@agentic/*` scope 在 npm 上的可用性必须发包前确认（`agentic` 一词与 LangChain 的 `AGENTS.md`、以及 npm 已有包可能冲突）。备选：`@<你的org>/harness-*`。

---

## 4. 导出规范（SDK 关键约定）

### 4.1 主用法（写进 SDK_GUIDE 的目标 API 形态）

```ts
import { createReactAgent, defineProfile, defineBundle } from '@agentic/core';
import { defineTool } from '@agentic/plugin-tools';
import { bundleStandard } from '@agentic/bundle-standard';
import { pluginMcp } from '@agentic/plugin-mcp';

const profile = defineProfile({ name: 'demo', extends: ['standard'],
  patches: [{ id: 'llm', use: 'anthropic', with: { model: '...' } }] });

const agent = createReactAgent({
  profile,
  tools: [searchTool],
  mcp: { servers: './mcp.json' },
  maxSteps: 16,
  on: { 'tools/pre-execute': approvalGate },   // waterfall：可改写/可拒绝
});

for await (const ev of agent.stream('任务')) { /* durable + live 混合投影 */ }
agent.dispose();                                // 全树 effect 逆序回滚，含 MCP 子进程
```

### 4.2 package.json exports 规则

- 每包必须显式声明 `exports`；`src/internal/*` 不出现在 exports 中；CI `check-exports` 校验
- `files` 白名单仅 `dist/`、`README.md`、`LICENSE`
- 双格式：`dist/index.js`（ESM，Bun/Node18+）+（可选）`.cjs` + `index.d.ts`（`tsc --emitDeclarationOnly`）
- `publishConfig.access: "public"`；根与 `apps/`、`examples/`、`profiles/` 保持 `private`
- 每包 `peerDependencies` 只声明 `@agentic/kernel`/`types` 版本轴，禁止 plugin 之间互为依赖

### 4.3 结构性约束（用脚本强制，不靠人自觉）

| 约束 | 守门脚本 |
|---|---|
| 内部实现不外泄 | `check-exports.ts` |
| 依赖方向单向、plugins 零互相 import | `check-deps.ts` |
| 每个 seam 三角色齐全、事件必须声明分发模式 | `check-seams.ts` |
| 事件契约与文档一致 | `verify-event-map.ts` |
| 一切注册必须经 effect（禁止裸 `setInterval`/`spawn`） | ESLint 自定义规则 `no-untracked-resource` |

---

## 5. Bun 工程配置

```jsonc
{
  "workspaces": ["packages/*", "packages/plugins/*", "packages/providers/*",
                 "packages/bundles/*", "apps/*", "examples/*"],
  "scripts": {
    "dev":            "bun run --cwd apps/playground src/main.ts",
    "repl":           "bun packages/cli/src/index.ts --profile cli",
    "test":           "bun test",
    "typecheck":      "tsc --noEmit -b",
    "build":          "bun run --filter '*' build",
    "check":          "bun scripts/check-exports.ts && bun scripts/check-deps.ts && bun scripts/check-seams.ts",
    "dump":           "bun packages/cli/src/index.ts dump-config --profile sdk",
    "mcp:doctor":     "bun packages/cli/src/index.ts mcp doctor",
    "release":        "bun scripts/release.ts"
  }
}
```

- `bunfig.toml`：test preload 加载 `.env`
- 依赖全走 `bun install`，`bun.lock` 入库
- 构建链只有 `bun build` + `tsc`（仅出声明），无 vite/webpack/ts-node

---

## 6. 与 v1 结构的差异（变更记录，方便你对照）

| v1 | v2 | 原因 |
|---|---|---|
| `core/src/agent/react-agent.ts` 内写循环 | 循环移到 `plugin-agent-loop-react`，`core` 只组装 | 循环本身必须是可替换插件（dsh 核心主张） |
| `core/src/middleware/` | 取消，改为 `waterfall` 事件监听者 + `agent.use()` 语法糖 | 固定中间件数组无法"短路/改写"，也不可撤销 |
| `core/src/prompt/` | 独立 `plugin-system-prompt`（`ctx.systemPrompt`） | 提示词装配是 seam，不是 core 私有物 |
| `providers/` 承担 LLM 抽象 + 实现 | 拆为 `plugin-llm`（seam）+ `provider-*`（实现） | Definition 与 Provider 分离是 seam 三角色的要求 |
| `tools/` 单包 | `plugin-tools`（注册表/管线）+ 内置工具归属待定（见 §7-3） | 注册表与工具内容不同生命周期 |
| `memory/` | `plugin-memory`（一期只定接口） | — |
| 无 MCP | `plugin-mcp` + `bundle-mcp` + CLI 三条 mcp 命令 | 你的新增要求，见 MCP_INTEGRATION.md |
| 无 profile/bundle | `core` 提供 `defineProfile/defineBundle/patch` + `profiles/` 模板 + `dump-config` | 分层可覆写 + 可诊断，是"不 fork 也能改"的落地形式 |
| 无 session log | `plugin-session`（append-only + deriveMessages） | 没有真相源就没有 fork/resume/审计（ARCHITECTURE §5） |
| 1 个 check 脚本 | 5 个结构守门脚本 | 约束越多越要靠工具而非约定 |

### 6.1 v2.1：案例研究回填（新增与加严，出处见 HARNESS_CASE_STUDIES.md）

| 变更 | 类型 | 出处 |
|---|---|---|
| 新增 `ctx.checkpoints` seam（**两个状态栈一起原子回滚：消息投影 + 世界状态**） | ➕ 新目录 `plugins/checkpoint/` | pi `/tree` 不回退文件的坑 + Shadow Git + Codex `ThreadRollback` |
| session log 从线性改**树**（事件带 `parentId`） | 🔧 加严 `plugin-session` | pi 会话树 |
| `ctx.inbox` 升为正式 seam（插话显式分类） | 🔧 加严 `plugin-agent-loop-react` | dsh inbox 机制 |
| 新增 `check-grepability.ts` | ➕ 守门脚本 | pi 的 grepability 判据 |
| 新增 `check-surface.ts`（工具数 / prompt token 上限） | ➕ 守门脚本 | pi 的三个赌注 |
| `defineTool` 参数校验失败**必须回填**为 `tool_result(isError)`，禁止 throw | 🔒 硬约束 | pi/TypeBox |
| 干预动作枚举化（`interrupt/compact/rollback/approve/shutdown` 走同一 Submission 通道） | 🔧 加严 `plugin-core` | Codex `Op` 枚举 |
| 纵深三道闸（policy → 路径约束 → argv/网络开关），passthrough 在 `doctor` 里标红 | 🔧 加严 `world` 插件组 | Codex 四道闸（裁剪版） |
| provider 传输降级路径（WS → SSE/HTTP） | 🔧 加严 `providers/*` | Codex `FallbackToHttp` |
| 多 Agent 默认用 **git worktree** 隔离 | ➕ `plugin-teams` 新增 `worktree.ts` | learn-claude-code s12 |
| `AGENTS.md` 明文“resist adding code to kernel” | ➕ 仓库根文件 | Codex 治理 |
| 知识进仓库：`AGENTS.md` / `skills/*.md` / `commands/*.md` 作为可 git 制品 | 🔧 加严 `plugin-system-prompt`/`plugin-commands` | pi 学徒模式 |

---

### 6.2 v2.2：内置工具与 Skills 回填（出处见 BUILTIN_TOOLS.md）

| 变更 | 类型 | 出处 |
|---|---|---|
| 内置 7+1 个原子工具，**零配置可用** | ➕ `plugin-fs`/`plugin-subprocess` 提为一期必做 | Deep Agents 内置工具表 |
| **工具改名对齐社区先验**：`fs_read` → `read_file`（+ `ls`/`write_file`/`edit_file`） | 🔧 修正 seam 表与示例 | Deep Agents / pi / dsh 均用简短动词 |
| `ctx.fs` 接口由 `read/write/list/stat` 改为 **七方法 + capabilities + routes** | 🔧 重定义 | Deep Agents `BackendProtocol` |
| **能力探测隐面**：`requires` 不满 → 工具从 schema 消失 | ➕ `plugin-tools/capabilities.ts` | 官方“auto hidden at request time” |
| 新 seam `ctx.skills` + 新包 `plugin-skills`（L1/L2/L3） | ➕ | Agent Skills 规范 + `SkillsMiddleware` |
| 新包 `plugin-planning`（`write_todos`，**opt-in**） | ➕ | Deep Agents 把它从默认改 opt-in（版本漂移教训） |
| artifact 卸载走 `ctx.fs` 路由（`/artifacts/`）而非独立存储 | 🔧 `plugin-context` | `/large_tool_results/` 与 CompositeBackend 路由 |
| **统一律：错误是数据，禁止 throw** | 🔒 硬约束 | 官方 *Do not raise exceptions* + pi 回填 |
| 扩展点定为 T1 数据 / T2 接口 / T3 事件，**不做 class 继承体系** | 🔒 设计裁决 | Deep Agents 同时给了 subclass **和** wrapper 两条路 |
| 顶层新增 `templates/dot-agentic/`（tools/skills/commands/mcp.json/AGENTS.md） | ➕ | pi 的 `.pi/` 约定 + 知识进仓库 |
| profiles 新增 `core8` / `coding` | ➕ | 把“工具数 ≤ 8”从抽象变成可执行默认面 |
| 守门脚本 7 → **9 个**（`check-tools.ts`、`check-skills.ts`） | ➕ | 能力面会变，必须用断言而不是文档 | 

---

### 6.3 v2.3：沙箱回填（出处见 SANDBOX.md）

| 变更 | 类型 | 出处 |
|---|---|---|
| `sandbox` 从二期**提到一期**（只做 `native`） | ➕ 范围变更 | `bash` 能否安全上面的前提 |
| 新增包 `@agentic/sandbox`（契约 + native + /testing） | ➕ | Claude Code 的 `@anthropic-ai/sandbox-runtime` 是现成参照物 |
| 新增包 `sandbox-container` / `sandbox-remote` / `sandbox-iso` | ➕ | gemini-cli 六后端 / E2B 事实标准 / oh-my-pi `pi-iso` 8 后端 |
| ★ **裁决：sandbox 派生 `ctx.fs` 与 `ctx.subprocess`**（而不是另开一套工具） | 🔒 设计裁决 | 避免“bash 在容器里、read_file 读宿主”的语义分裂 |
| **两种正交隔离分开建模**：`sandbox`（能碰什么）vs `checkpoints`/`worktree`（落在哪里） | 🔒 设计裁决 | “7 大开源 Agent 安全篇”：dsh/omp 的语义根本不是一回事 |
| `probe()` + `SandboxReport` + `sandbox/probe` durable 事件 | ➕ `plugin-sandbox` | letta “先跑探针再收紧”；我们补“事实必须可回放” |
| **fail-closed 为默认**（不静默降级），降级需显式写且进 log | 🔒 硬约束 | Codex/dsh fail-closed vs Claude Code 默认降级 |
| `deny` 语义＝**目录缺席**（tmpfs / 不 bind），不是 EACCES | 🔧 `native/*` | letta `--tmpfs`：“不可写”等于递一张“秘密在此”的地图 |
| 策略覆盖算法三条 + carve-out 不得为 deny 祖先的**装配期断言** | 🔒 硬约束 | bwrap last-mount-wins 静默暴露；Claude Code “deny 在宽 allow 内仍生效” |
| 网络单独一面：`none`/`proxy`（域名白名单 + 本地 CONNECT 代理）+ `<network>` 段进 prompt | ➕ `sandbox/net-proxy.ts` | Codex `network-proxy`（MITM 与凭据注入→二期） |
| **敏感文件硬过滤**进 `plugin-policy`（一期就做，与 OS 沙箱正交） | ➕ | kimi `tools/policies/sensitive.ts`（便宜、有效、能直接拿来用） |
| 升权：`allowUnsandboxed` 默认 **false**；走 `approve` Submission 通道；封闭词汇 `granted\|denied\|cancelled\|unavailable` | 🔒 硬约束 | Claude Code 逃生舱太软 + dsh 升权词汇 |
| 哨兵 env `AGENTIC_SANDBOX=1` 防嵌套重入；`--die-with-parent` + Windows 进程树回收 | 🔧 `plugin-sandbox` / `mcp/transport/stdio` | letta；复用现有实现不写两遍 |
| `capabilities.snapshot` 接入 `checkpoints` 的 `messages\|workspace\|world` 三档 scope | 🔧 `plugin-checkpoint` | “续跑是沙箱价值的一半”（各家共识） |
| 顶层新增 `templates/dot-agentic/{sandbox.json,sandbox.Dockerfile}` | ➕ | 与 `mcp.json` 同级的对外约定面；`.gemini/sandbox.Dockerfile` |
| profiles 绑定沙箱默认档（sdk/core8=`workspace-write`、scout=`read-only`、headless=容器） | 🔧 `profiles/*` | letta “按数据敏感度给不同默认值” |
| examples 新增 **H-sandbox** 组 7 例（70–76） | ➕ | 把“隔离到底隔了什么”变成可跑的样例 |
| 守门脚本 9 → **10 个**（`check-sandbox.ts`） | ➕ | 策略与参数生成必须靠断言，不靠文档 |
| 明确 **Windows 本机一期无 OS 沙箱**（走 WSL2 或 Docker）；写进 `SECURITY.md` | ⚠️ 诚实边界 | opencode “声明不提供什么”范本；Codex 拒 WSL1 |

---

## 7. 拍板点（**v2.4 全部已定：均按各项括号里的建议值采纳**，逐条裁决见 [ARCHITECTURE.md](./ARCHITECTURE.md) §9）

1. **包粒度**：一期 plugin 独立多包（结构最纯，包数 ~23） vs 单包多子路径 `@agentic/plugins/*`（包少、发布简单，但"移除一个能力"不够干净）。**我倾向独立多包**，但这直接决定工作量与发布复杂度，需要你定。
2. **一期能力集**：`ARCHITECTURE.md` §3 的 7 个必做 seam 之外，`fs` / `subprocess` / `sandbox`（native 档）/ `skills` / `planning` 已因“内置工具零配置 + 沙箱”提为一期——**要不要再加东西？（我的建议：不加，一期就到这里）**
3. **内置工具归属**（v2.2 已给建议，待你确认）：**工具跟着能力走** —— `ls/read_file/write_file/edit_file/glob/grep` 归 `plugin-fs`、`bash` 归 `plugin-subprocess`、`task` 归 `plugin-teams`，`plugin-tools` 只留注册表/管线/装饰器（否则 seam 的 Consumer 角色会被拆散）。剩下的真问题只两个：是否允许用户把内置工具从面上摘除（我的答案：**允许摘、不允许改名**，改名会破坏模型先验），以及**无沙箱时 `bash` 默认是否在面上**——v2.3 已给答案：**不在**（fail-closed，`capabilities.shell` 探不到就直接隐面，见 SANDBOX.md §4.4）。
4. **Session log 一期必做？**（不做则无 fork/resume/审计，也失去"模型可见=已记录"这条最有价值的不变量）**强烈建议做，JSONL 即可。**
5. **MCP 一期包官方 SDK 还是自研最小客户端？**（见 MCP_INTEGRATION §11-1）
6. **是否现在预留 `evals/`**：harness 的价值主张就是"同模型换 harness 分数不同"，没有评测目录就无法证明设计有效。**建议一期就建 `evals/`（哪怕只有 3 个用例）。**
7. **CJS 兼容是否必要**：只发 ESM 可显著简化 exports 与构建。
8. **scope 名**：`@agentic/*` npm 可用性待核。
9. **v2.1 新增的 4 个取舍**：`ctx.checkpoints` 一期是否必做（我建议必做，否则 fork/并行不成立）、session 树一期开多少、三道闸默认强度（allow-all vs workspace-write）、worktree 是否作为 teams 默认隔离单位——**详见 [HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md) §7**。
10. **v2.2 新增的 4 个取舍**：内置工具改名（`fs_read` → `read_file`，我建议改）、`task` 是否算 core 第 8 个工具、skills 规模墙阈值（建议 30）、`toolFromFunction` 是否对外公开——**详见 [BUILTIN_TOOLS.md](./BUILTIN_TOOLS.md) §7**。
11. **v2.3 新增的 4 个取舍**：沙箱默认档（我建议 `workspace-write` + fail-closed）、升权逃生舱默认关还是默认开（我建议**关**）、容器后端是否进一期（我建议**进**，否则你在 Windows 本机无法验证）、`WorkspaceView` 接口是否一期就抽象出来（我建议**是**，否则 `checkpoints` 与 `teams` 会写成两套）——**详见 [SANDBOX.md](./SANDBOX.md) §8**。

---

## 8. 实施顺序（已开工）

1. ~~你批注 §7 十一个点 → 我改文档定稿~~ ✅ 已按建议值定稿（v2.4）
2. `bun install` + workspaces 骨架 + 10 个守门脚本先跑通（无业务代码）
3. 内核 `@agentic/kernel` + 其单测（effect 逆序回滚、依赖重启、层级隔离）—— 这是全局地基，必须先绿
4. `@agentic/sandbox` 契约 + `native` 后端 + `fakeSandbox`（因为它卡着 `bash` 能不能上面上，**必须在内置工具之前就位**）
5. `plugin-agent-loop-react` + `plugin-session` + `provider-openai` → 跑通第一个 `createReactAgent().run()`
6. `plugin-fs` / `plugin-subprocess` 由 sandbox 派生 → 跑通 `examples/70-native-workspace-write`
7. `plugin-mcp` + `mcp doctor` → 接第一个真实 MCP server
8. 回头补 `SEAM_CATALOG.md` / `EVENT_MAP.md` / `SECURITY.md`（从代码生成，避免手写漂移）
