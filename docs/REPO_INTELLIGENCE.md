# 让 Agent 真正看懂项目：代码理解层设计（教学篇）

> 直接回应你的痛点：**"我之前设计的 Agent，问它项目是什么，它只读 README。"**
> 这不是提示词问题，是**工具面问题**。下面先讲清根因，再给出可落地的分层方案。
> 调研输入：2026 年代码理解工具三条技术路线横评（tree-sitter 符号图 / 向量检索 / Agent 自主探索）、Aider repo map、Sourcegraph SCIP、`codebase-memory-mcp`（162 语言 AST + Hybrid LSP + 图查询 + MCP 17 工具）、dsh 的 `dsh-code-intel`。

---

## Part 1 根因诊断：为什么它只读 README

Agent 的行为 = **它能看到的动作空间里最便宜的那条路**。

```
它只有：read_file / grep / list_dir
你问：「这个项目是干什么的？」
最便宜的路径：读 README（1 个文件，几千 token，看起来"有名字有描述"）
   → 它诚实交付了「一个最省力的答案」
```

三条必然结论：
1. **没有结构化工具，就没有结构化理解。** `grep ChargeCard(` 能找到 3 处直接调用，但**找不到** `PlaceOrder`——因为它的代码里根本没有 `ChargeCard` 这个词；要找到它得先意识到 `CardCharger` 实现了 `Charger` 接口，再去搜谁调了 `.Charge(`。**这类问题的本质是图遍历，不是文本匹配，也不是向量相似度。**
2. **"只读 README"是工具缺失的症状**。给它 `get_architecture()` / `trace_path()` / `who_calls()`，它才有第二条路可走。
3. **向量检索解决不了"关系"**。它返回的是"相似片段列表"，不是"A 节点与 B 节点之间存在调用关系"这种结构化事实。**信息检索 ≠ 语义理解。**

> 所以第一步不是改 prompt，是**改变它的成本景观**：让"正确的深度分析"比"读 README"更省事。

---

## Part 2 目标定义：什么叫"看懂了项目"

把这五件事当作可验收清单（这也是我们工具设计的输入）：

| 能力 | 具体问题 | 靠什么实现 | 只读 README 能否做到 |
|---|---|---|---|
| **T1 符号定位** | `ChargeCard` 定义在哪、签名是什么 | AST 符号索引 | ✗ |
| **T2 语义搜索** | "处理支付重试的逻辑在哪" | 向量 + FTS（认 camelCase） | ✗ |
| **T3 影响分析** | 改这个函数影响哪些模块/测试 | 调用图反向遍历 | ✗ |
| **T4 架构理解** | 分层、入口、数据流、外部依赖 | 图 + 聚类 + 摘要产物 | △ 只能拿到作者想让你看到的那部分 |
| **T5 历史与意图** | 为什么这么写、哪里在腐烂、评审意见是什么 | git log / blame / PR 评论 | ✗ |

**T5 是"深度"的关键分水岭。** README 描述的是**意图**，代码描述的是**现实**，git 历史描述的是**两者如何偏离**。一个真正懂项目的 Agent，能说出"这里 README 说支持多租户，但 `tenant_id` 在 `tx.go` 这条路径上没透传，2 个月前那次重构引入的"。这句话只可能来自 T3+T5 的交叉。

---

## Part 3 分层方案（我们的 `plugin-repo-intel`）

### 3.1 四层索引流水线

```
① 解析层   Tree-sitter（162 语言，容错：代码有语法错也能出部分 AST；增量解析 100ms 级）
                    ↓ 语法兜底
② 精化层   Hybrid LSP 式类型感知（把「同名调用」精确到具体定义）
           ← 两层互相兜底：少①只能支持十几语言；少②调用图全是歧义边
③ 建图层   点 = 符号（函数/类/接口/路由/配置项/测试）
           边 = calls / implements / overrides / imports / handles-route / tested-by / configures
④ 检索层   三套并存：
           (a) 结构查询：类 Cypher 只读子集 → query_graph / trace_path
           (b) 语义搜索：内置 embedding（本地，不需 API key）
           (c) 全文检索：FTS5 + 驼峰/下划线切分器 ← 最不起眼但最关键
                        （代码里是 getUserById，通用分词器当成一个词，认不出就搜不到）
```

**存储**：本地 SQLite（内存优先构建 → 一次性 dump → 释放内存），图谱快照可提交进仓库供团队共享增量，避免每人重跑全量。规模参考：Linux 内核（2800 万行 / 7.5 万文件）官方自报 3 分钟全量索引（**未经独立复现，仅作数量级参考**）。

**索引 pass 要拆开、可增量**（这是工程可行性的关键，别写成一个巨型函数）：
`definitions` `calls` `usages` `routes` `config` `infra`（Dockerfile/K8s 清单也是图的组成部分，不是纯文本）`dbt/tests` `gitdiff` `githistory` `complexity` `cross-repo` + `incremental` + `delta`。

### 3.2 暴露给模型的工具（Consumer 角色）

对齐 seam 的三角色：`ctx.repoIntel` 是 Definition，`plugin-repo-intel` 是 Provider，**下面这批工具就是 Consumer**。

| 工具 | 用途 | 备注 |
|---|---|---|
| `repo_overview(root?)` | 项目身份卡：语言/框架/入口/分层/外部依赖/**README 与现实的不一致点** | **取代"读 README"**，见 §4 |
| `search_graph(query)` | 符号级结构搜索 | 首选入口 |
| `get_symbol(name)` | 定义 + 签名 + 直接邻居 | |
| `trace_path(from, to, {direction, depth})` | 调用链/反向影响链 | 回答"改了会波及谁" |
| `query_graph(pattern)` | 类 Cypher 结构查询 | 高级用户的逃生舱 |
| `get_file_outline(path)` | 文件内符号骨架（不读全文） | 便宜 |
| `read_code(symbol \| path, range)` | 按需取原文 | **唯一会读原文的口子** |
| `history_of(path \| symbol)` | git log/blame 摘要 + 变更频率 | T5 |
| `detect_changes(base)` | git diff → 受影响符号集 | CI/评审用 |
| `index_status()` / `coverage()` | 索引进度与覆盖率 | 覆盖率不足要**明说**，别让 Agent 自信地基于 30% 索引作答 |

**工具面裁剪（重要）**：全量开放 17 个 vs 侦察模式只开 8 个 —— 成熟实现明确给出两套 profile，理由是 **"模型面对 17 个工具时选错的概率更高"**。我们的 `scout` profile 只给：`repo_overview` `search_graph` `trace_path` `get_symbol` `get_file_outline` `index_status` `coverage` `read_code`。

### 3.3 关键取舍：**先查图，后读文件**（成本证据）

一份公开实测：**5 个结构化查询，逐文件搜索约 41.2 万 token，知识图谱约 3400 token，省 99.2%**（同一工具自报，未经独立复现，但数量级差异足以定策略）。

```
❌ Agent 自主探索：list_dir → read README → grep → read 12 个文件 → （token 爆炸，且看到的是局部）
✅ 图优先：repo_overview → search_graph → trace_path → 只在需要证据时 read_code 取 2 段
```

**这条纪律要在 harness 里强制，而不是建议**：`tools/pre-execute` waterfall 上监听 `read_code`，若本次 turn 还没调用过 `repo_overview`/`search_graph`，就**返回替代结果**："先调用 repo_overview 或 search_graph 定位，再按需读取；当前盲目读取会消耗 ~X tokens"。这是把"正确路径"变成"最便宜路径"的具体手段。

---

## Part 4 `repo_overview` 怎么产出"架构理解"（而不是文件清单）

这是你最关心的那一环。它不是 `tree` 命令的翻译，而是**一次带验收标准的分析**：

```ts
interface RepoOverview {
  identity: { remote?: string; fingerprint: string; languages: Lang[]; frameworks: string[] };
  buildRun: { install: string; test: string; lint: string; dev: string; ci: string; source: string };
  entrypoints: { kind: 'cli'|'http'|'main'|'job'; symbol: string; note: string }[];
  architecture: { layers: {name, symbols, dependsOn}[]; dataFlow: string; boundaries: string[] };
  inventory: { symbols: number; files: number; tests: number; coverageOfIndex: number };
  // ↓ 这三块才是"深度"，README 永远给不出
  intentVsReality: { docClaim: string; docSource: string; reality: string; gap: string; confidence: number }[];
  hotspots: { area: string; churn: number; ownedBy: string[]; riskNote: string }[];
  conventions: { naming: string; errorHandling: string; testing: string; evidence: string[] }[];
  openQuestions: string[];        // ★ 诚实列出"我没搞清楚的"
  indexCoverage: number;          // ★ 低覆盖时整份产物标为「初步」
}
```

**产出方式（成本与质量折中，别指望一次调用搞定）：**

```
阶段 A 结构扫描（无 LLM，纯图）   → layers / entrypoints / inventory / hotspots（churn 来自 git）
阶段 B 约定归纳（LLM，小上下文）  → 每个约定只喂 3 个代表性片段 + 符号名，归纳并【必须附证据 ref】
阶段 C 文档对照（LLM）           → README/注释/类型声明 与 实际图 交叉 → intentVsReality
阶段 D 缺口自省（LLM）           → 列出 openQuestions，标注 indexCoverage
产物 → 存为 artifact + 写入 .agentic/repo-overview.json（缓存 + 团队共享）
```

**三条质量约束：**
1. **每条结论必须带 `evidence`**（文件:行 或 符号 id）。没有证据的架构判断，与幻觉无异，用户无法验证。
2. **`intentVsReality` 是核心差异化能力**。README 说"支持 X"，图上找不到支撑路径 —— 这一条一出现，用户立刻知道"这 Agent 真看了代码"。
3. **缓存 + 增量失效**：`repo-overview.json` 带 `fingerprint`；文件变更走 `pipeline_delta` 增量更新，只在结构性变化（新增入口、删除模块、依赖大改）时重跑阶段 B/C。**每次会话都重跑全量分析 = 不可用的延迟与成本。**

---

## Part 5 "根据一句话推导出一个项目"：把理解前置成阶段

这是你提的另一个目标（一句话 → 做出项目）。正确结构是**三阶段门禁**，而不是"边想边写"：

```
① Orient（定位，scout profile，只读工具，无写权限）
   repo_overview → 找相似实现 → trace_path 确认扩展点 → 产出「改动计划」
   ▸ 门禁：计划必须含【具体文件+符号+改动性质+影响面+测试位置】，纯散文计划不予放行
        ↓ policy: require-approval（计划交人确认，Dots 的 require approval 模式）
② Execute（analysis profile，开放写工具 + bash）
   按计划逐步改，每步一次 checkpoint；偏离计划要显式声明偏离理由（防止目标漂移）
        ↓
③ Validate（跑 test/lint/build + detect_changes 影响核对）
   失败 → 回到②修正；连续两轮无进展 → 回到①重规划（LOOP_ENGINEERING §3.2）
```

**为什么必须分三阶段**：单阶段循环里，模型的第一次猜测会立刻变成写操作，错误代价从"一句废话"升级成"改坏 12 个文件"。分层后 **Orient 的错误只损失一次规划 token**。

**"一句话 → 项目"从零生成的场景**（仓库尚不存在）同理：把 Orient 阶段换成「先产出 SPEC + 目录设计 + 依赖选型 + 验收清单，交人确认，再生成」，并规定**生成骨架后立刻建索引**，让后续步骤进入图优先模式。

---

## Part 6 "越了解项目"如何与"越了解用户"接上

代码理解层天然产出**项目作用域记忆**，与 [CONTEXT_BUDGET_MEMORY.md](./CONTEXT_BUDGET_MEMORY.md) §3/§4 打通：

| 代码侧发现 | 沉淀成的记忆（predicate 示例） | 后续效果 |
|---|---|---|
| `conventions` | `project.style.error-handling` | 生成代码自动贴合，不用每次提醒 |
| `buildRun` | `project.build.cmd` / `project.test.cmd` | 跳过"先猜命令再失败重试"的浪费轮次 |
| 用户反复手改同一类问题 | `user.pref.no-{pattern}`（纠正流） | 下次不再生成该类写法 |
| 被接受的 PR 结构 | `project.pr.template` | 自动按团队模板出 diff |
| `hotspots` | `project.risk-{area}` | 改到高风险区自动加测/加审批 |
| `intentVsReality` | `project.debt-{gap}` | 主动提醒，而不是照着过时 README 写 |

**工程要求**：`plugin-repo-intel` 与 `plugin-memory` **不互相 import**，各自 `ctx.repoIntel` / `ctx.memory` 提供与消费；由 `bundle-*` 组装决定谁在用谁（对齐 ARCHITECTURE 的插件零耦合纪律）。

---

## Part 7 一期落地范围（务实版）

| 项 | 一期 | 说明 |
|---|---|---|
| Tree-sitter 符号索引 | ✅ | 走现成 CLI/库，不自研解析器；**先支持 TS/JS + Python + Go** |
| 调用图（含类型精化） | △ 简化 | 一期用"导入图 + 同名调用启发式 + LSP 可选增强"，歧义边如实标注 |
| 语义 + FTS 检索 | ✅ 复用 | 直接接一个现成代码索引 MCP server（`mcp__code-intel__*`），**比自己实现划算** |
| `repo_overview` 四阶段产物 | ✅ | 这是我们的差异化层，成本主要在阶段 B/C 的 LLM 调用，做好缓存 |
| git churn / history / detect_changes | ✅ 基础版 | `history_of` + `detect_changes` 先用，`cross-repo` 二期 |
| 增量索引 + 快照共享 | ✅ | `.agentic/graph.db.zst` 提交策略写进文档 |
| 只读 scout 工具面 + 图优先门禁 | ✅ | §3.3 的强制手段，一期就做，收益最大成本最低 |
| 跨服务/跨仓库链接、DBT/K8s pass | ❌ 二期 | |

> **最省事的第一步**：先接一个现成的代码图谱 MCP server（走我们已有的 `plugin-mcp`），把它的工具面裁成 scout 8 个，再加我们自己的 `repo_overview` 四阶段产物。**这样两周内就能明显看到"不再只读 README"的效果**，而完整的索引器可以之后替换——因为 Consumer 侧只有工具契约，替换 provider 正是 seam 的意义。
