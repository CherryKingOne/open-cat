# 内置原子工具与 Skills（Deep Agents 调研 + 本项目的方案）

> 回答你的四点：**① 基础工具怎么做到不用手动定义；② Skills 怎么用、用户怎么自定义；③ 工具是写死还是让人继承类；④ 工具格式怎么定。**
> 配套：[ARCHITECTURE.md](./ARCHITECTURE.md) §3、[SDK_SURFACE.md](./SDK_SURFACE.md) §1、[HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md)、[SANDBOX.md](./SANDBOX.md)（`bash` 工具能不能上面上、看见什么目录，全由它定）
> 状态：**v2.4 已定稿**（§7 的 4 个点已全部按建议值采纳，裁决记录在 [ARCHITECTURE.md](./ARCHITECTURE.md) §9-9）

---

## 0. 先把两条信息更正一下（你说信息不准确，确实有两处）

**① "原工具" = 原子工具（built-in / harness tools）**，即 harness 自带、用户不给也存在的那批最小能力。下文统一叫**内置原子工具**。

**② Deep Agents 是什么**（我核实的版本，`langchain-ai/deepagents`，约 28.6K star，自称 *"the batteries-included agent harness"*）：

- 它是 LangChain 生态的**最上层**：`LangGraph`（运行时：状态/检查点/流式/中断恢复）→ `LangChain create_agent`（框架：模型+工具+循环）→ **`Deep Agents`（harness：把编排决策做成默认值）** → `LangSmith`（观测/评测）。
- 官方 FAQ 的定位原话：Deep Agents 是 `create_agent` 之上**"更具主见"（more opinionated）**的 harness。灵感来源是 Claude Code——团队想搞清楚 Claude Code 为什么通用，把那些设计提炼成可编程框架。
- **有官方 JS/TS 版（Deep Agents.js）**，核心 API 与 Python 对齐。所以它是能在我们 Bun/TS 语境下直接对照的少数样本之一。
- 它的主仓库里 `profiles/` 放**按模型/供应商调优**的配置，`libs/code/` 是预构建的终端编码 agent（带 TUI、远程沙箱、持久记忆）。

**当前版本的内置工具清单**（以官方 tools 文档为准）：

| 工具 | 来源 | 备注 |
|---|---|---|
| `ls` `read_file` `write_file` `edit_file` `glob` `grep` | Filesystem middleware 注入 | 全部委托给 backend，工具本身是薄壳 |
| `delete` | 同上，**可选** | backend 不支持时**该工具自动对模型隐藏** |
| `execute` | 仅 sandbox / local-shell backend | 实现了 `SandboxBackendProtocol` 才出现 |
| `task` | subagent middleware | 派生子 agent 处理委派任务 |
| `write_todos` | `TodoListMiddleware`，**opt-in** | ⚠️ 见下方"版本漂移" |

> ⚠️ **一个很有信息量的细节**：早期版本（以及大量中文教程）都说 `write_todos` 是每个 deep agent 默认自带；**当前文档已经改成"要用 `TodoListMiddleware` 显式 opt-in"**。
> 这说明了三件事：**(a) "默认能力面"是会变的，而且是产品级决策；(b) 二手教程在这个话题上极不可靠；(c) 我们的能力面必须受 CI 断言与 changelog 约束（`check-surface.ts` / `check-tools.ts`），不能靠文档里写死的一句话。** 这就是本项目为什么要给 core profile 的工具数和 prompt token 数上硬断言。

---

## 1. Deep Agents 是**怎么**做内置工具的（机制，不是清单）

这六条才是值得抄的东西。每条都给"它怎么做 / 为什么这样 / 我们怎么办"。

### 1.1 工具由 middleware 注入，而不是 agent 写死

`create_deep_agent(backend=..., skills=..., tools=[...])`——你给的是**配置与实现**，工具面由 middleware 按配置生成。传了 `skills` 路径才有 `SkillsMiddleware`；换了 `backend` 类型，`execute` 会自己出现或消失。

**为什么**：内置工具的存在与否应该是一个**由配置推导出来的结果**，不是一个手写的数组字面量。手写数组意味着"实现换了但面没换"，于是模型看见一堆必然失败的工具。
**我们怎么办**：`bundles` / `profiles` 声明 provider，工具面由 §1.3 的能力探测生成，core 里没有 `BUILTIN_TOOLS = [...]` 这种常量清单。

### 1.2 工具只是薄壳，真正的能力在 Backend（协议）

官方要求自定义 backend 实现 `BackendProtocol`，七个方法：

| 方法 | 签名（官方口径） | 说明 |
|---|---|---|
| `ls` | `(path) -> LsResult` | 列目录 |
| `read` | `(file_path, offset=0, limit=2000) -> ReadResult` | **默认限行**，防上下文爆炸 |
| `write` | `(file_path, content) -> WriteResult` | 写 |
| `edit` | `(file_path, old_string, new_string, replace_all=False) -> EditResult` | 查找替换，**不是 diff/patch** |
| `glob` | `(pattern, path=None) -> GlobResult` | 按模式找文件 |
| `grep` | `(pattern, path=None, glob=None) -> GrepResult` | 搜内容（字面串） |
| `delete` | `(file_path) -> DeleteResult` | **可选** |
| `execute` | （仅 `SandboxBackendProtocol`） | 多这个方法 → `execute` 工具才存在 |

现成实现：`StateBackend`（默认，thread 内有效）/ `FilesystemBackend`（本地磁盘，需 `root_dir` 绝对路径）/ `StoreBackend`（跨 thread 持久）/ `ContextHubBackend`（LangSmith Hub 仓库）/ `CompositeBackend`（路由）/ sandbox（Daytona 等）/ **自定义**（官方直接给了一个 `S3Backend` 例子）。

**为什么**：`tools` 是"模型词汇表"，`backend` 是"能力"。分开之后，**换存储/换沙箱/加审计都不用碰工具定义**。
**我们怎么办**：这正是我们的 seam 三角色（Definition / Provider / Consumer）。内置工具的 Consumer 层极薄，实质契约在 `ctx.fs` / `ctx.subprocess` 的 Definition 上。**结论：把 `ctx.fs` 的接口名对齐这套七个方法**（见 §2.6），因为这套形状是被反复验证过的最小充分集。

### 1.3 能力探测决定模型可见的工具面 ★ 本轮最重要的借鉴

官方原话（`delete` 那一行）：

> *If the backend does not support deletion, **the tool is automatically hidden from the model at request time**.*

**为什么这条了不起**：它把"权限/能力"问题从**运行时拒绝**前移成了**装配时不可见**。模型看不见一个它做不到的动作，于是：不会浪费一步去调用它、不会拿到一条错误再猜、不会有"绕不过去的诱惑"。这比"调用了然后返回 permission denied"省下整个 step 的钱和一次行为漂移风险。
**我们怎么办**：provider 必须声明 `capabilities: Set<string>`；工具注册表在每次装配（`agent/request` 之前的 prompt 装配阶段）按能力过滤。**"能力缺失 = 工具面缺失"** 写进内核规则，并由 `check-tools.ts` 断言"任何被 profile 声明的工具，其依赖能力必须在该 profile 的 provider 里可满足"。

> 对照 dsh：dsh 是靠 `scout` profile 手工裁成 8 个工具；Deep Agents 是靠能力探测自动隐。我们**两个都用**——手工声明 profile（可读、可审计）+ 能力探测兜底（防止声明与实现漂移）。

### 1.4 失败是数据，不是异常

官方一句极其朴素的硬规定：

> *Always return **structured result types with an `error` field** for failure cases. **Do not raise exceptions.***

**为什么**：一次 raise 就打断循环（这正是 pi 那条"参数幻觉直接把整个会话崩掉"的同一个死法）。而带 `error` 的结构化结果会作为 tool result 回到模型眼前，模型能读、能改、能换路。
**我们怎么办**：把它和 pi 的"校验失败回填"合并成**一条统一律**（见 §5.2）：**错误是数据，不是控制流**。

### 1.5 权限在 backend 之前评估，且给了两条扩展路线

- **声明式规则**：`permissions=[FilesystemPermission(operations=["write"], paths=["/policies/**"], mode="deny")]`，官方明确说"**应用于内置文件系统工具，且在 backend 被调用之前评估**"。
- **超出路径 allow/deny 的逻辑**（限流、审计、内容检查）：官方给的路线是 **"subclassing or wrapping a backend"**——例子有 `GuardedBackend(FilesystemBackend)`（继承）和 `PolicyWrapper(BackendProtocol)`（包装）。
- 另外反复强调：`FilesystemBackend` **务必配 `virtual_mode=True` + `root_dir`**（拦 `..`、`~`、越界绝对路径），并强烈建议开 HITL middleware。

**为什么值得注意**：Deep Agents 在这件事上**同时提供了继承和包装两条路**——这直接回答了你的第 3 问（见 §4）。连他们都不把继承做成唯一路径。
**我们怎么办**：路径规则 = `ctx.policy` 的声明式规则数组（一期就要有）；更复杂的 = **包装 provider（`with*` helper）**；两条都优于改工具代码。

### 1.6 文件系统是唯一的"工作介质"，连内部数据都写进虚拟 FS

Deep Agents 会**自动**把大工具结果驱逐到 `/large_tool_results/`、把对话历史写进 `/conversation_history/`——都是 backend 里的路径。配合可路由的 `CompositeBackend`：

```python
backend=CompositeBackend(
    default=StateBackend(),                 # 内部数据留在 thread 状态
    routes={"/proj/": FilesystemBackend(root_dir="...")},  # 项目文件落真盘
)
```

官方 tip 说得很直白：**单独用 `FilesystemBackend` 时，这些内部文件会真的写到你 `root_dir` 的磁盘上，把 agent 产物和你的项目文件混在一起。**

**为什么这条对我们特别有用**：这正是 [CONTEXT_BUDGET_MEMORY.md](./CONTEXT_BUDGET_MEMORY.md) §2.3 的 artifact offload，但 Deep Agents 的做法更进一步——**卸载介质 = 同一个 FS seam**，于是"模型可以 read_file 把它读回来"是天然成立的（不需要 `ctx.artifacts.get` 这种第二套 API）。
**我们怎么办**：
1. artifact 卸载走 `ctx.fs` 的路径约定，而不是独立存储层：**`/artifacts/<callId>.txt`**（我们的命名，不叫 `/large_tool_results/`，见下）。
2. **一期就内建路由**（内部路径 ≠ 用户项目路径），不要复制 Deep Agents 那个"默认混在一起"的坑。这条要写进 `ctx.fs` 的 Definition：`routes` + `default`，且 `dumpConfig()` 打印实际生效映射。

---

## 2. 我们的内置原子工具面（回答你的第 1 点）

**一句话设计目标**：`createReactAgent()` **一个 tool 都不传，就有 7 个原子工具可用**；这 7 个由各能力插件贡献，**core 里没有工具清单常量**。

### 2.1 内置清单与默认参数（一期）

| 工具 | 参数（要点） | 只读 | 破坏性 | 备注 |
|---|---|---|---|---|
| `ls` | `path` | ✅ | — | 目录列表 |
| `read_file` | `path`, `offset=0`, **`limit=2000`** | ✅ | — | **默认限行**（照抄 §1.2，大文件是上下文爆炸头号来源） |
| `write_file` | `path`, `content` | ❌ | ✅(新建/覆盖) | 走 policy ask |
| `edit_file` | `path`, `old_string`, `new_string`, `replace_all=false` | ❌ | ✅ | **唯一匹配才算成功**，多处匹配返回可行动错误 |
| `glob` | `pattern`, `path?` | ✅ | — | |
| `grep` | `pattern`, `path?`, `glob?` | ✅ | — | 一期先字面串，二期正则 |
| `bash` | `command`, `timeoutMs?`, `cwd?` | ❌ | ❓ | 由 `ctx.sandbox` 的 `probe()` 报的 `capabilities.shell` 决定是否出现（**探不到笼子能力就 fail-closed：`bash` 直接不在面上**）；`scout` profile 下笼子＝`read-only`，只能跑白名单只读命令 |
| `task` | `prompt`, `agent?` | ❌ | ❌ | 派生子 agent；是否算 core 见 §7-2 |

### 2.2 ★ 命名要改：从 `fs_read` 改成社区约定的 `read_file`（本轮真实的设计修正）

现状：`ARCHITECTURE.md` 的 seam 表里写的是 `fs_*` 工具。我建议**改掉**，依据是**四个成熟项目在这批工具上的命名高度收敛，而且都是简短动词**：

| 项目 | 文件读 | 文件写 | 改文件 | 跑命令 |
|---|---|---|---|---|
| Deep Agents | `read_file` | `write_file` | `edit_file` | `execute` |
| pi | `read` | `write` | `edit` | `bash` |
| Codex | — | — | `apply_patch` | `exec_command` |
| dsh | 简短动词（白皮书明确要求） | | | |

**理由**：模型是被这些名字训练出来的，**改名等于浪费先验**——`fs_read` 这种带前缀的名字对模型没有任何额外信息，还多占 token。dsh 白皮书也明确"模型实际可调用的工具名是简短动词"。
**代价**：`fs_read` 前缀原本是为了避免和 MCP 工具重名。裁决：**冲突改用命名空间清洗解决**（与 MCP 同一套 `mcp__server__tool` 规则），而不是给内置工具加前缀。内置名进白名单，`check-tools.ts` 校验。
**`bash` 而不是 `execute`**：三个候选里（`bash`/`execute`/`exec_command`）选 `bash`——最短、且与 pi/dsh 一致。

同理，`edit_file` 的参数名**必须**用 `old_string` / `new_string`，不用 `diff` / `patch`：模型对这套参数名有强先验，且这个形状天生支持"失败后重试"（唯一性检查），而 patch 格式一旦行号漂移就整体失败、模型很难自愈。

### 2.3 "基础工具无需手动定义"的具体含义

| 层次 | 用户要做什么 | 我们保证什么 |
|---|---|---|
| 开箱 | **什么都不写** | core profile 拿到 §2.1 的 7 个；带默认参数与安全标记 |
| 微调 | `createReactAgent({ tools: { bash: { timeoutMs: 60_000 } } })` | 内置工具可**改配置**（超时/限行/根目录/是否暴露），不必重新定义 |
| 增补 | `tools: [myTool]` | 与内置工具同一条注册路径、同一套契约（没有特权工具） |
| 替换 | patch 掉 `ctx.fs` 的 provider | 内置 `read_file` 立刻指向你的实现，**工具名与描述不变** |
| 关闭 | profile 里不声明 `subprocess` | `bash` 直接从模型可见面上消失（§1.3） |

### 2.4 core 能力面正好是 8 个

[HARNESS_CASE_STUDIES.md](./HARNESS_CASE_STUDIES.md) §5-9 立了断言：`core` profile 工具数 ≤ 8。§2.1 这张表正好 **7 + `task` = 8**，不是硬凑：文件系统的六个最小动作 + 一个 shell，是任何"能干活"的 agent 的下界，`task` 是并行的下界。
- `write_todos`（planning）、`search_tools`（渐进披露）、`repo_*`（代码理解）全部落在 **`coding` / `research` profile**，另立预算（建议 ≤14）。
- 新增 profile 项：`profiles/coding/`（fs + shell + skills + planning + repo-intel）。

### 2.5 输出治理（内置工具默认就有，用户工具同规则）

- 任何工具结果 > `artifactRefThreshold`（CONTEXT_BUDGET_MEMORY §1 定为 4096 token 等价位）→ 全文写 `/artifacts/<callId>.*`，上下文里只留 **摘要 + 路径 + 取回方式**。
- 因为 artifact 就在 `ctx.fs` 里（§1.6），模型可以用**它已经会的** `read_file(offset, limit)` 局部读回来——**不需要新工具，也不需要新 API**。这是我认为 Deep Agents 这套设计里最省的一笔。
- 内置工具全部声明 `maxOutputBytes`，`grep`/`glob` 默认截断并明确告知"结果被截断，可用 path/glob 收窄"。

### 2.6 对 `ctx.fs` 接口的具体修正（回填 ARCHITECTURE）

原定义 `read/write/list/stat` → 改为对齐 §1.2 的七方法形状（`ls/read/write/edit/glob/grep/delete?`）+ **能力声明**。理由：这套形状是被 Deep Agents（Python + JS）、dsh、pi 三家独立验证过的**最小充分集**；`stat` 不在模型需要的词汇表里（模型要的是"有哪些文件/内容是什么"，不是 inode 元数据），删掉。

---

## 3. Skills：怎么用、用户怎么自定义（回答你的第 2 点）

### 3.1 采用开放规范，不发明格式

用 **[Agent Skills](https://agentskills.io/specification) 规范**（Anthropic 发起的开放格式）：一个技能 = **一个目录 + 一个 `SKILL.md`**（YAML frontmatter: `name` + `description`，正文是给 agent 的指令），可选 `scripts/` `references/` `assets/`。Deep Agents 明确遵循此规范。

**为什么不自己定格式**：这条生态已经统一了——Claude Code、Codex、Deep Agents、mattpocock/skills 都用 SKILL.md。**用户已有的技能文件应该直接能用**，这是"继承生态"而不是"发明格式"。我们只做增强层（§3.3）。

### 3.2 三级渐进披露（Deep Agents 的机制，我们照做）

| 级别 | 加载什么 | 何时 | 谁负责 |
|---|---|---|---|
| **L1 Metadata** | `name` + `description` | Agent 启动，**每个**已配置技能 | `SkillsMiddleware` → 注入 system prompt |
| **L2 Instructions** | `SKILL.md` 正文 | 技能被激活时 | **模型自己调 `read_file`** |
| **L3 Resources** | `scripts/` `references/` `assets/` | 正文引用到时 | 模型按需 `read_file` / `bash` |

> **注意 L2 没有新机制**——命中后就是模型读文件。这是极省的一笔：技能加载复用 §2.1 的 `read_file`，不需要 `activate_skill` 工具、不需要新的注入通道、不需要新的权限面。我们照抄。

官方给的量级约束（我们进 CI）：**frontmatter 保持精简**、**`SKILL.md` 正文 < 5000 token**。

### 3.3 我们加的三条（Deep Agents 没做的）

**(a) 规模墙提前。** LangChain 对自家 Kai 的披露：技能目录叠到 **约 150 个**时，frontier model 的选择质量已经下降，于是他们转向"检索/分类器预筛 → 再让 LLM 终选"。
→ 我们把阈值定为 **30**（而不是等 150）：超过即自动切换为预筛路径（用小模型/分类器做召回，见 LOOP_ENGINEERING 的 Decisions API 一节），`check-surface.ts` 同时断言 L1 段的 token 上限。
→ 附带好处：**30 这个阈值是可以在 evals 里用数据调的**，因为下一条 (b) 落了数据。

**(b) 命中必须可评测。** 新增事件 `skills/activated`（durable）：记录候选集、命中项、任务摘要、该 turn 的终态。没有这一条，技能目录只能靠感觉维护——加一个技能会不会让别的技能被误选，永远没人知道。

**(c) 安全分级（这是我们比 Deep Agents 严的地方）。**
- `SKILL.md` 的 frontmatter 与正文是**指令** → 进入 prompt 即视为**不可信内容**参与注入风险分级；第三方源的 `description` 同样要标（prompt injection 就藏在"一句话说明"里）。
- `scripts/` 是**代码** → **默认不自动执行**；技能正文写"请运行 scripts/fetch.py"时，必须走 `bash` 的 policy ask + `trust` 声明。
- 落盘：技能清单哈希进 session log，变更告警（与 MCP 的 tool poisoning 机制同一条）。

### 3.4 多源与优先级

```
~/.agentic/skills/          # 用户全局
<repo>/.agentic/skills/     # 项目级（进 git，可 code review，团队共享）
bundle 声明的 skills         # 可分发能力包
profile 显式指定的路径        # 最高
```
同名后者覆盖前者；`agentic config dump`（`dumpConfig()`）必须能打印**每个生效技能的实际来源路径**——否则用户无法调试"我改的 skill 为什么没生效"。
Deep Agents 有个反直觉细节要写进我们的校验：**`skills` 路径必须指向"包含技能目录的目录"**，直接指到某个含 `SKILL.md` 的技能目录本身**不会被加载**。这种"静默不加载"的约定必须在 `check-skills`/CLI 里报错提示，不要让用户猜。

### 3.5 用户自定义工具的三条路（第 2 问的后半）

| 路线 | 形态 | 适合 | 成本 | 我们的落点 |
|---|---|---|---|---|
| **进程内代码** | `.agentic/tools/xxx.ts` 默认导出 `defineTool(...)` | 私有 API、需要本机权限、要求低延迟 | 写 TS，随仓库走 | 自动加载，**默认关闭需 `trust`**（pi 的坑二） |
| **纯提示（技能）** | `.agentic/skills/<name>/SKILL.md` | 方法论、流程、领域知识；**不碰代码** | 只写 Markdown | L1/L2/L3 机制 |
| **外部服务** | MCP server | 跨语言、第三方、需进程隔离、要复用给别的 agent | 起一个 server | `plugin-mcp`，命名 `mcp__server__tool` |

判据一句话：**"要新逻辑"选 1，"要新知识"选 2，"要新进程/新语言"选 3。**（与 HARNESS_CASE_STUDIES §4-4 的 A/B 判据一致）

---

## 4. 写死 vs 继承类？（回答你的第 3 点）

**裁决：三层扩展点；主路径是"数据 + 接口"，不做 `class BaseTool` 继承体系。**

| 层 | 扩展方式 | 覆盖场景 | 形态 | Deep Agents 里的对应物 |
|---|---|---|---|---|
| **T1 工具级** | `defineTool({...})` 传进 `tools` | ~90% 的日常需求 | **数据 + 函数** | 传 plain function / `@tool` / dict 给 `tools=` |
| **T2 能力级** | **实现 interface**（`FsBackend` / `ShellBackend` / `LlmAdapter`），或**包装**别人的 provider | 换存储、换沙箱、接 OSS、加审计 | **interface + provider 注册** | 实现 `BackendProtocol`（例：`S3Backend`）；`PolicyWrapper` |
| **T3 行为级** | waterfall 监听者 / `ctx.effect` / patch 一行 | 拦截、改写参数、加策略、埋点 | **事件 + 配置** | middleware、`permissions=[...]` |

**为什么不上 class 继承（三条硬理由，不是口味问题）**：

1. **TS 的结构类型系统下，继承没有契约收益。** `interface FsBackend` + 任意对象实现就能保证兼容；class 继承带来的 `this` 绑定、字段遮蔽、原型链，在纯函数式 provider 里只有成本。Java/C# 那种"继承获得多态"在 TS 里由结构类型免费提供了。
2. **和内核铁律冲突。** 我们的两条铁律是"一切注册皆可撤销（`ctx.effect`）"和"模型可见即已记录（session log 可重放）"。这要求注册物是**可 dump、可 patch、可序列化的数据描述**——`dumpConfig()` 要能打印出每一行的来源与配置。class 实例带隐藏状态与闭包，做不到可复现的 dump/patch。
3. **生态信号一致。** Deep Agents 用 Protocol、pi 用 `defineTool` 对象、MCP 用 JSON schema、Codex 用枚举——**没有一家把抽象基类做成用户主路径**。Deep Agents 确实在文档里出现过 subclass（`GuardedBackend(FilesystemBackend)`），但注意：**同一页官方同时给了 `PolicyWrapper` 的包装路线**，说明连他们也不把继承当唯一答案，而当作"内部实现复用的便利"。

**但保留"像继承的体验"**（很多人问继承，其实是想要"少写重复"）：

```ts
// 装饰 helper，而不是基类。动词规范见 SDK_SURFACE §2 的 with*
// 落点是 plugin-tools 的子路径（对外只开这一个装饰器面，见 SDK_SURFACE §2.2）
import { withGuards, withRetry, withCache } from '@agentic/plugin-tools/operators';

export const deployTool = withGuards(defineTool({ name: 'deploy', /* ... */ }), {
  deny: [{ operations: ['*'], paths: ['**/prod/**'] }],   // 复用 §1.5 的声明式规则形状
});
```

**"写死"这一半的答案**：内置工具是**默认值，不是不可动的常量**。§2.3 表最后一行——patch 掉 `ctx.fs` 的 provider，`read_file` 的名字和描述不变、实现整个换掉。这就是"内置但不写死"。

**类型推导（你的第 3 问真正想要的收益在这里）**：`defineTool` 由 schema **反推参数类型**，`run(args)` 里的 `args` 自动带类型（pi 的一份定义三产出）。可选糖 `toolFromFunction(fn)`——从签名 + JSDoc 推 schema，对齐 Deep Agents 的低摩擦体验（他们原话：*"infers the tool schema from the function signature and docstring"*）。
> ⚠️ 但我建议**提供而不推荐**：把 description 交给 JSDoc，等于把提示词质量交给了注释——**描述质量直接决定模型选不选得对**，而注释改一个字 = schema 变 = 稳定前缀变 = 缓存击穿（见 CONTEXT_BUDGET_MEMORY §1.3）。对外发布的关键工具应该显式 `defineTool`，让描述成为被 review 的、带版本的契约面。

---

## 5. 工具格式契约（回答你的第 4 点）

### 5.1 `ToolDefinition` 的字段面（定死这个格式）

| 字段 | 必填 | 类型要点 | 缺失后果 |
|---|---|---|---|
| `name` | ✅ | snake_case，**短动词**；`^[a-z][a-z0-9_]{0,47}$`；内置名进白名单 | 模型选错 / 与 MCP 冲突 |
| `description` | ✅ | 一句话说清 **做什么 + 何时用 + 上限**；建议 40–300 字符 | 太短选不出，太长费 token |
| `parameters` | ✅ | JSON Schema，**strict**：`additionalProperties: false`、`required` 齐全 | 参数幻觉 |
| `annotations` | ✅ | `{ readOnly, destructive, openWorld, idempotent }` | policy 无法自动决策 |
| `requires` | ✅* | 依赖的能力名（如 `['fs']`）→ 驱动 §1.3 的隐藏机制 | 声明了却必然失败 |
| `run(args, ctx, signal)` | ✅ | 返回 `ToolResult`，**不得 throw**（§5.2） | 会话崩 |
| `describeError(e)` | 建议 | 错误 → **可行动的下一步** | 瞎重试撞满 maxSteps |
| `validate(args)` | 建议 | 前置校验；失败也走 §5.2 回填 | 钱花了才发现参数错 |
| `cost` | 建议 | `{ estTokens, estLatencyMs, concurrentSafe }` | 预算层无法提前拦 |
| `maxOutputBytes` | 建议 | 超阈值 → `/artifacts/` 卸载 | 上下文爆炸 |
| `examples?` | 可选 | 2–3 个正例 + **1 个不该用的反例** | 误用 |
| `version?` | 可选 | schema 变更参与清单哈希 | 无法发现 tool poisoning |

### 5.2 统一律：**错误是数据，不是控制流**

这条把 pi（校验失败当 tool result 返回）和 Deep Agents（*do not raise exceptions*）合并成一条：

```ts
type ToolResult =
  | { ok: true;  data: unknown; artifact?: string; truncated?: boolean }
  | { ok: false; error: string; retryable?: boolean; next?: string };  // ★ next = 给模型的下一步建议
```

**三条禁止**：
1. **禁止 `throw` 打断循环**——业务失败一律 `ok: false`。（与 SDK_SURFACE §1.3 第 2 条同一条律）
2. **禁止把原始 stack trace / HTTP 响应体丢回模型**——它读不出可行动信息，只会重试。
3. **禁止"不支持"作为运行时错误返回**——不支持就**不在面上**（§1.3 的能力探测）。一个模型看得见但一定失败的工具，是纯粹的注意力税和成本。

**唯一例外**：真正的编程错误（`undefined is not a function`、类型系统被绕过）——抛到 `tool/internal-error` 事件、终止本 step、标 `finish: 'error'`。**不要伪装成工具结果喂给模型**，那是在教它"这个 agent 坏了，我多试几次"。

### 5.3 注册与生命周期

- 注册即 `ctx.effect`：卸载逆序回滚（内核铁律，不变）。
- 每次装配产生的**工具清单哈希**进 session log；清单变化（尤其 MCP 来源）→ 告警。
- 工具是数据描述 → 可以被 `dumpConfig()` 打印、被 patch 替换、被 evals 快照。这是不上 class 的直接回报。

### 5.4 CI 守门（新增 `scripts/check-tools.ts`）

断言：命名正则 / 内置名白名单 / description 长度区间 / strict schema 齐全 / `annotations` 齐全 / **每个工具至少一条 `ok: false` 路径的单测** / `requires` 的能力在声明该工具的 profile 里可满足 / core profile 工具数 = 声明值。

### 5.5 一个完整示例：`edit_file`（最能体现这些契约）

```ts
export const editFile = defineTool({
  name: 'edit_file',
  description:
    '在已有文件中做精确字符串替换。old_string 必须在文件中唯一匹配，否则失败；' +
    '确需批量替换时显式设置 replace_all。新建文件请用 write_file。',
  parameters: {
    type: 'object', additionalProperties: false,
    required: ['path', 'old_string', 'new_string'],
    properties: {
      path: { type: 'string' },
      old_string: { type: 'string', description: '要被替换的原样文本，不做归一化' },
      new_string: { type: 'string' },
      replace_all: { type: 'boolean', default: false },
    },
  },
  annotations: { readOnly: false, destructive: true, openWorld: false, idempotent: false },
  requires: ['fs'],
  cost: { estTokens: 900, estLatencyMs: 40, concurrentSafe: false },
  maxOutputBytes: 8_192,
  async run({ path, old_string, new_string, replace_all }, ctx) {
    const r = await ctx.fs.edit({ path, oldString, newString, replaceAll });  // 委托给 provider，不碰磁盘
    if (r.error) return fail(r.error, nextFor(r.error));   // ★ §1.2：provider 返回 error，不抛
    return ok({ replaced: r.count, path });
  },
  describeError(e) {
    if (e.code === 'NOT_UNIQUE')   return 'old_string 命中多处，请扩大它前后上下文使其唯一后重试';
    if (e.code === 'NOT_FOUND')     return 'old_string 未命中，请先 read_file 确认当前内容（可能已被你或他人改过）';
    if (e.code === 'PERMISSION')    return '该路径不在写权限范围内，请把改动申请交给审批或改到工作区内';
    return `不可恢复：${e.message}`;
  },
});
```

注意三条：**(a)** 工具不自己 `fs.readFile`，一切走 `ctx.fs`（否则 §1.6 的路由与沙箱会失效——`ctx.fs` 的默认 provider 是由 sandbox 派生的，直连磁盘等于绕过笼子）；**(b)** 失败信息是"下一步动作"而不是状态描述；**(c)** `idempotent: false` + `destructive: true` → policy 默认 ask。

---

## 6. 一期落地清单（按依赖顺序）

| # | 做什么 | 归属 | 验收 |
|---|---|---|---|
| 1 | **sandbox 契约 + `native` 后端（macOS Seatbelt / Linux bwrap）+ `fakeSandbox`**；`probe()` 报 capabilities，`openFs()`/`openShell()` 能派生后端 | ★ 新包 `@agentic/sandbox` + `@agentic/plugin-sandbox` | `check-sandbox.ts` 通过；无可用后端时 **`bash` 不在模型面上**（fail-closed，而不是调了才报错） |
| 2 | `ctx.fs` 接口改成七方法形状 + `capabilities`；实现 `StateFs` / `LocalFs(virtual_mode)`，**默认 provider 由 sandbox 派生** | `plugin-fs` | 单测 + 换 backend 不改工具名即可跑通；**换 container 后端后 `read_file` 与 `bash` 看见同一个世界** |
| 3 | 内置 7+1 工具，命名对齐（§2.2），Consumer 薄壳 | `plugin-fs` / `plugin-subprocess` / `plugin-teams` | `createReactAgent()` 零配置能读写改列搜 |
| 4 | **能力探测隐面**：`requires` → 装配时过滤（数据源＝各 provider 的 `capabilities`，含 sandbox 的 `probe()`） | `plugin-tools` | 给不支持 delete 的 backend 时，`delete` 不出现在 schema 里 |
| 5 | `ToolResult` 契约 + **禁止 throw**（§5.2） | `types` + `plugin-tools` | `check-tools.ts` 的失败路径单测断言 |
| 6 | 输出驱逐到 `/artifacts/` + `CompositeBackend` 式路由（内外部隔离） | `plugin-context` + `plugin-fs` | 项目目录里不出现 harness 内部文件 |
| 7 | `ctx.skills`（L1/L2/L3 + 多源优先级 + trust 分级 + `skills/activated` 事件） | 新包 `plugin-skills` | 已有 Agent Skills 目录零改动可用 |
| 8 | `ctx.planning`（`write_todos`，**opt-in**，带状态） | 新包 `plugin-planning` | 默认 core profile **不含**它 |
| 9 | `.agentic/{tools,skills,commands,mcp.json,sandbox.json}` 约定目录 + `trust` 开关 | `core` + `cli` | 未 trust 的仓库里不自动加载任何代码；但 **`sandbox.json` 是收紧不是放权，不需 trust** |
| 10 | `check-tools.ts` + `check-skills.ts` + `check-sandbox.ts` | `scripts/` | CI 阻断 |

> 三类目录变更是本轮造成的：第 1 项是**新增包组**（sandbox 四个包）、第 7/8 项是**新增包**、第 9 项是**新增顶层约定**——这是目录结构必须更新的原因，见下。
> ⚠️ **为什么 sandbox 拍在第 1 位而不是最后**：它卡着 `bash` 能不能上面上、以及所有文件工具看见哪个世界。先做内置工具后补沙箱，会先训练出一批"假定无笼子"的行为与测试，返工成本最高。

---

## 7. 四个拍板点（**v2.4 已全部采纳建议值**）

1. **内置工具改名**（`fs_read` → `read_file` 等，§2.2）：我建议改。这会同步修正 `ARCHITECTURE.md` 的 seam 表与示例。
2. **`task` 算不算 core 的第 8 个工具**：算 → 多 agent 开箱可用但子 agent 成本不可见；不算 → core 收到 7 个，`task` 归 teams opt-in。**我倾向"算，但默认预算封顶"**（子 agent 共享父预算且单列）。
3. **skills 规模墙阈值 30** 是否合适（Deep Agents 的观测是 ~150 才开始退化）；越低越省 token，但预筛会引入召回误差。**建议一期 30，二期用 evals 数据调。**
4. **`toolFromFunction`（签名 + JSDoc 推 schema）是否对外公开**：我建议**公开但文档标注"不建议用于对外发布的关键工具"**（理由见 §4 末尾）。

> 另有 4 个属于沙箱的待拍板点（默认档位 / 逃生舱 / 容器是否进一期 / CoW 工作区视图是否一期就抽象），列在 [SANDBOX.md](./SANDBOX.md) §8。
>
> ✅ **定稿结果（逐条）**：1 改名为 `read_file`/`write_file`/`edit_file`/`bash`（已同步修正 `ARCHITECTURE.md` 与 `PROJECT_STRUCTURE.md`）；2 `task` 算第 8 个，默认预算封顶；3 规模墙 30；4 `toolFromFunction` 公开但不推荐。

---

## 8. 参考资料（本轮核实过的来源）

- **Deep Agents**：`langchain-ai/deepagents`（PyPI `deepagents 0.7.x`，官方 README 指向 JS/TS 版 Deep Agents.js）；官方文档 `docs.langchain.com/oss/python/deepagents/{tools,backends,skills,sandboxes,permissions,human-in-the-loop,customization,overview}`；`BackendProtocol` / `SandboxBackendProtocol` 参考页；`SkillsMiddleware` 参考页。关键事实：内置工具表（含 `delete` 自动隐藏、`execute` 仅 sandbox）、七方法签名与默认 `limit=2000`、*"Always return structured result types with an error field. Do not raise exceptions."*、`/large_tool_results/` 与 `/conversation_history/`、`virtual_mode=True` 安全提示、`write_todos` 已改 opt-in（`TodoListMiddleware`）、`MCPAdapter.list_tools()` → `tools=`。
- **Skills 规范**：agentskills.io specification（`SKILL.md` + frontmatter `name`/`description` + `scripts/references/assets`）；datawhalechina `deepagents-in-action` ch07（三级 Progressive Disclosure 图与多源优先级）；LangChain 对 Kai 的 ~150 技能选择退化披露；Anthropic Skills 的 Discovery/Activation/Execution 三段式。
- **对照样本**：dsh 白皮书（工具名用简短动词、`tools/list` 缓存）；pi（4 工具 + <1000 token + 进程内 `.pi/tools`）；Codex（`exec_command`/`apply_patch`、prompt 与模型配套调优）。

> 本文对 Deep Agents 的描述以**官方文档当前版本**为准；中文教程里仍大量流传"write_todos 默认自带"的旧说法（见 §0 版本漂移注）。
