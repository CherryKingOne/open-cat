# 上下文阈值、Context 关联与 Memory 设计（教学篇）

> 回答三个点名问题：**阈值在哪儿？Context 怎么关联？Memory 怎么设计（长期/短期）？**
> 数字以 2026-09 各家公开规格为锚（下列 provider 数字取自 OpenAI 公开定价页与系统卡，**会随时变化，务必做成可配置，不要写死在代码里**）。

---

## Part 1 阈值到底在哪儿：五层，不是一个数

新手把"阈值"理解成"上下文窗口 200K，用到 180K 就压缩"。这只是第 1 层。真实的 harness 要在**五层**上分别设线，而且**越靠后的层越致命**。

### 1.1 五层阈值全景

| 层 | 阈值 | 以 GPT-6 Astra 为例的公开数字 | 撞线的表现 | 正确反应 |
|---|---|---|---|---|
| **① 窗口层** | 模型 context window / max output | **1M token 上下文，128K 最大输出** | 直接报 `context_length_exceeded` | 压缩 + 卸载（**不是重试**） |
| **② 价格阶梯层** | 超过某长度单价跳档 | **prompt ≤ 272K：$10/$50 每百万；> 272K：$20/$75（翻倍）** | 不报错，**账单翻倍**，且没有任何提示 | 把长上下文**主动控制在 272K 以下**，或改成检索式喂料 |
| **③ 缓存层** | 前缀命中才有折扣 | **cached read ≈ $1/M（约为输入价的 1/10）** | 不报错，**每轮多花 10 倍** | 前缀必须**稳定 + 只追加**（见 §1.3） |
| **④ 吞吐/速率层** | RPM / TPM / TPS | 见 LOOP_ENGINEERING §4，按 provider 配额 | 429 | RPM→退避；**TPM→降 token** |
| **⑤ 质量层（最容易被忽略）** | 有效注意力远小于窗口 | 经验值：**长上下文的中段信息召回显著劣化**（lost-in-the-middle），且 token 越多模型越容易"迷路" | 不报错、不花钱异常，只是**答案变差** | 主动裁剪 + 关键信息**放到头尾**，而不是塞满窗口 |

> **第②③层是我希望你重点记住的**：它们**不抛异常**，只会安静地多收你 10~20 倍的钱。绝大多数"Agent 成本失控"事故的根因在这里，而不在窗口溢出。

### 1.2 具体建议阈值（我们的默认值，全部可配）

```ts
const contextBudget = {
  // ① 窗口层：给输出和安全边际留足空间，永远不要顶到窗口上限
  maxPromptTokens:  floor(model.contextWindow * 0.75),   // 1M 模型 → 750K
  reserveForOutput: min(model.maxOutput, plannedOutput + 4096),

  // ② 价格阶梯层：留 10% 余量，避免边界抖动导致整轮跳档
  softCostStep: 245_000,        // < 272K 的实际防线（示例值，按 provider 配）

  // ③ 缓存层
  stablePrefixMinRatio: 0.6,    // 前缀至少占 60% 且跨轮不变，否则缓存基本白搭

  // ④ 速率层
  tpmHeadroom: 0.8,             // 只用到配额的 80%

  // ⑤ 质量层
  workingSetTokens: 24_000,     // 真正进模型的"活跃工作集"，远小于窗口
  artifactRefThreshold: 4_096,  // 工具输出超过这个长度就卸载成 ref，不进上下文
};
```

**核心观念转变**：
> 窗口是**上限**，不是**目标**。harness 的本事是**用 24K 的活跃工作集完成需要 200K 信息的任务**——靠的是检索、引用、卸载和压缩，不是靠窗口大。

### 1.3 缓存与"稳定前缀"：一个设计决定，牵动全部结构

这是我认为本项目**最重要的一个工程洞察**，也正好解释 dsh 里那几个看起来莫名其妙的概念为什么存在。

缓存命中要求**前缀逐字节相同**。因此：

1. **system prompt 必须"冻结在头部"**，不能每轮重新渲染出细微差异（时间戳、UUID、字典序不稳定都会击穿缓存）。dsh 专门规定"第一个被准入的 step 先占住 system 头部，即使用户消息为空"，就是在守这条。
2. **历史只能追加，不能改写**。一旦改写中间消息，其后所有前缀失效。→ 所以 dsh 要求插件"不许改 log，只能注册 pure message projection"。
3. **工具 schema 变化 = 前缀变化**。所以 dsh 区分"能缓存的路由：工具更新追加在缓存历史之后"和"不能缓存的路由：在第一个 system 节点合并"。
4. **需要换上下文基底时，显式开一个新的 request series**（dsh 的 `startsRequestSeries` / `request/header`），而不是偷偷改：因为改了就知道缓存已失效，那不如趁机把该合并的信息合并掉，避免后续每一轮都白付钱。

**因此我们的 harness 必须有这三个概念**（一期内置，不做完也要留接口）：
- `requestSeries`：一组共享同一前缀的请求，series 内缓存有效；
- `assembly digest`：本次装配的哈希，用于诊断"为什么缓存没命中"；
- `frozen history`：派生给模型的历史一旦被某 series 使用即视为不可变。

> 排查成本问题的第一句话应该是：**"我的缓存命中率是多少？"** 如果接近 0，你付的是全额输入价，长上下文 Agent 基本不可能便宜。

---

## Part 2 Context 怎么"关联"：三个维度

"Context 怎么关联"这个问题其实是三件事，分开答：

### 2.1 纵向关联：历史如何流进本轮（身份链）

```
Task ▸ Session ▸ Turn ▸ Step ▸ Call ▸ Artifact
        │        │       │      │       │
        │        │       │      │       └─ 卸载的大块内容（工具输出、文件、截图）
        │        │       │      └─ 幂等键、参数哈希（跨轮重复检测）
        │        │       └─ requestId / requestSeries（缓存边界）
        │        └─ turnId（fork/resume 的锚点）
        └─ sessionId（真相源容器）
```

每一层都持有父层 id，于是任何一次模型请求都能回答："这段上下文来自哪个 turn 的哪个 step、哪个工具的哪次调用"。**这条链是可观测性、成本归因、断点续跑和审计的共同基础**——缺了它，你事后无法解释"它为什么这么说"。

### 2.2 横向关联：本轮上下文由哪些块拼成（装配 = 分段 + 位置策略）

上下文不是"历史消息数组"，而是**分段装配的结果**。我们的 `ctx.systemPrompt` 与装配层按段贡献：

```ts
type AssemblySection = {
  name: string;            // 'persona' | 'rules' | 'repo-intel' | 'memory' | 'task-state' | 'tool-schema' | 'history' | 'scratchpad'
  content: ContentPart[];
  priority: number;        // 超预算时的丢弃顺序（小的先丢）
  volatile: boolean;       // ★ 是否允许出现在稳定前缀里（true 则必须排在易变区）
  position: 'head' | 'tail'; // ★ 质量层策略：关键信息放头尾
};
```

**两条硬规则：**
- `volatile: true` 的段（当前时间、实时 token 数、本轮观察）**必须排在稳定前缀之后**，否则击穿缓存；
- 超预算时按 `priority` **从低到高丢弃**，并且**丢弃动作本身要落 log**（否则模型以为自己看到了全部）。

### 2.3 引用关联：大块内容怎么"不进上下文但可用"（Artifact / Offload）

这是"用 24K 工作集干 200K 的活"的具体机制：

```
工具输出 40K token
   ↓ 超过 artifactRefThreshold
落盘为 artifact：ref = art:step-42:tool-call-7
   ↓ 模型只看到
"结果已存 art:step-42:tool-call-7（40,213 tokens，摘要：登录失败源于 token 过期，涉及 3 个调用点）"
   ↓ 需要时
模型调 read_artifact(ref, { range, query }) 取回局部
```

摘要要**结构化**（长度、类型、关键结论、可选取回方式），不能只写"内容过长已省略"——后者模型完全不知道该不该取回。dsh 的做法（长截图 OCR、上下文压缩面板）也是这个思路：**卸载不等于丢弃，卸载是把寻址权交回模型。**

### 2.4 压缩（Compaction）的次序，和它必须付出的代价

```ts
// 触发：估算 token 超 softCostStep 或工作集超限
const order = [
  'drop-low-priority-sections',   // 1. 先丢低优段（最便宜）
  'trim-tool-results',            // 2. 截断旧工具输出（保留结论，丢过程）
  'summarize-old-turns',          // 3. 折叠远端 turn 为摘要（近端保持原样）
  'archive-and-reindex',          // 4. 归档 + 交给 memory 检索层兜底
];
```
- **永远保留**：persona/rules、当前用户请求、最近 N 轮原文、已验证事实（带 ref）。
- **压缩必须开新 request series**（前缀已变），并在 log 里记一次 `system/message` 更新。
- **压缩是有损的**：所以第 4 步必须有 memory 检索兜底。**没有 memory 层的压缩 = 失忆**。

---

## Part 3 Memory：短期与长期

### 3.1 先把四层"记忆"分清楚（混用是设计失败的起点）

| 层 | 生命周期 | 载体 | 谁负责 | 典型内容 |
|---|---|---|---|---|
| **工作记忆** | 单个 step | `AgentState` + scratchpad | loop | 目标、游标、未解决条件、最近调用签名 |
| **情景记忆（短期）** | 单个 session | session log + artifacts | `ctx.sessions` | 这次任务做了什么、结论、失败尝试 |
| **语义记忆（长期）** | 跨 session | 结构化记忆库 | `ctx.memory` | 用户偏好、项目事实、术语、约定 |
| **程序性记忆** | 跨 session | 技能 / 规则包 | 插件 + Skill | "这个项目怎么跑测试""怎么发 PR" |

**大多数人只做了第 2 层（历史记录），就以为自己有 memory 了。** 真正的 memory 是第 3、4 层：**能跨 session 改变默认行为**。

### 3.2 写入路径（写比读难，因为要治理）

```
turn 结束（settlement）
   ↓ 抽取（用一个受约束的决策模型，别用自由生成 —— 对齐 Decisions API 思路）
候选记忆 {kind, predicate, value, evidence_refs, scope, confidence}
   ↓ 去重：与既有记忆做语义 + 结构化匹配
   ├─ 完全重复 → 只提升 confidence 与 last_seen_at
   ├─ 冲突 → 不覆盖！两条都留，标 superseded_by / disputed，进入冲突队列
   └─ 新增 → 落库 + 建索引（向量 + FTS）
   ↓ TTL / decay：长期不用降权；用户显式纠正 → 立即改写
```

**存储与寻址设计：**

```ts
interface MemoryEntry {
  id: string;
  kind: 'preference' | 'project-fact' | 'relation' | 'procedure' | 'glossary';
  predicate: string;            // ★ 结构化键：如 'project.build.cmd'、'user.comm.style'
  value: unknown;               // 类型化载荷
  evidence: ArtifactRef[];      // ★ 必须可溯源：这条记忆从哪次观察得来
  scope: { userId?: string; projectHash?: string; branch?: string };  // 隔离边界
  confidence: number;           // 0..1，随强化/衰减变化
  createdAt: number; lastSeenAt: number; ttl?: number;
  status: 'active' | 'disputed' | 'superseded';
  embedding: Float32Array;      // 语义检索
}
```

**三条我认为不可妥协的纪律：**
1. **没有 evidence 的记忆不许写入。** 否则你无法在出错时回滚，也无法解释"它为什么认为我喜欢这样"。
2. **冲突不覆盖，标注并存。** 用户说"用 pnpm"、lockfile 是 `bun.lock` → 两条都记，降 confidence，必要时在 turn 里问一次。**静默覆盖会造成"记忆漂移"**，表现为 Agent 行为莫名其妙地反复变化。
3. **记忆必须按 scope 隔离。** 项目 A 的约定泄漏到项目 B 是用户最反感的失效模式。`scope.projectHash` 由仓库身份（remote url + 结构指纹）决定，见 [REPO_INTELLIGENCE.md](./REPO_INTELLIGENCE.md)。

### 3.3 读取路径：召回 → 预算 → 定位

```ts
async function recallInto(ctx: AgentContext, query: Query, budget: TokenBudget): AssemblySection[] {
  const hits = await Promise.all([
    semantic(query, { topK: 12 }),                     // 向量
    lexical(query, { topK: 8 }),                       // FTS5（认 camelCase/snake_case）
    keyed(query.scope, ['project.build.cmd', 'user.comm.style']), // 结构化键：偏好类必须必取
  ]);
  const fused = rrf(hits)                              // 倒数排名融合
    .boost(m => m.confidence * decay(m.lastSeenAt))
    .filter(m => !m.disputed || mustAsk(m));
  return pack(fused, budget)                           // 按预算截断
    .place('head', pinnedPreferences)                  // ★ 偏好/规则放头（缓存友好 + 抗中段衰减）
    .place('tail', relevantFacts);                     // ★ 本次相关事实放尾
}
```

要点：
- **偏好类记忆走"必取"通道**（按结构化 key 直接拉），不能指望语义检索召回"用户喜欢简洁回复"这种无查询词的条目。
- **位置重要**：关键约束放头尾，避免 lost-in-the-middle。
- **注入要显式标注来源**：让模型知道"这是记忆，不是用户本轮说的"，否则它会把旧记忆当新指令，产生"用户明明没这么说"的幻觉。**这一点是 Memory 相关 bug 的头号来源。**

### 3.4 遗忘：不做遗忘的记忆库一定会变成污染源

```
decay: confidence *= exp(-Δt / halfLife)
archive: confidence < 0.3 且 90 天未 seen → 移出主索引（保留原始证据，可回滚）
consolidate: 同一 predicate 多条 → 周期性合并为 1 条 + 保留证据链
user-visible: 必须提供"看/改/删"界面 —— 不可见、不可删的记忆在用户眼里等于不信任
```

---

## Part 4 越用越懂用户：反馈闭环怎么落到工程上

"越了解项目就越了解用户"这句话，拆开是**三条独立的数据通路**，缺一条都做不出来：

| 通路 | 采什么 | 变成什么 | 何时生效 |
|---|---|---|---|
| **纠正流** | 用户改了什么（reject/编辑 diff）、说了"不对，应该…" | `preference` 记忆 + 程序性规则 | 下一轮 |
| **行为流** | 哪些结果被采纳、哪些被丢弃、任务实际怎么完成的（改了哪些文件、跑了什么命令） | 高频动作统计 → 快捷路径 / Skill 候选 | 下次同类任务 |
| **项目流** | 仓库结构、约定、依赖、CI、历史（git log/PR 评审意见） | `project-fact` + repo 知识图谱 | 每次装配注入 |

**关键工程要求：**
1. **反馈必须是"可验证信号"，不是模型自评。** 用户接受 = 正反馈；用户编辑了你的输出 = 强负反馈（且 diff 本身带着正确答案）。这两类信号成本极低但价值最高，**必须在 harness 层显式采集**（`ctx.sessions` 记录 `accepted` / `edited` / `rejected` 三种 settlement）。
2. **记忆写入门槛要高，读取门槛要低。** 宁缺毋滥：一条错误记忆会长期污染行为，且用户很难发现根因（"它最近怎么老这样"）。
3. **每次注入都留痕**，并支持 `agent.explain()`："这轮我引用了这 4 条记忆，来源分别是…"。没有可解释性，用户遇到一次奇怪行为就再也不信你了。
4. **冷启动要有分级**：新仓库先建立项目流（一次索引），纠正流随使用累积；不要第一天就尝试"个性化"。

---

## Part 5 一页速查（贴墙上）

```
阈值     : 窗口 ≠ 目标 | 价格阶梯与缓存【不报错，只烧钱】 | 5 层分别设线
缓存     : system 冻结在头 + 历史只追加 + volatile 段靠后 + 换基底开新 series
装配     : 分段 + priority + position(head/tail) + volatile 标记 + 丢弃要落 log
卸载     : 超阈值 → artifact ref + 结构化摘要 → read_artifact 取回局部
压缩     : 丢段 → 截断工具输出 → 折叠远端 turn → 归档（memory 兜底）→ 开新 series
记忆分层 : 工作 | 情景(session) | 语义(长期) | 程序性(技能)
记忆写入 : settlement → 受约束抽取 → 去重/冲突并存 → 必带 evidence + scope + confidence
记忆读取 : 必取(偏好 by key) + 语义 + 词法 → RRF → 预算截断 → 头尾定位 → 标注"这是记忆"
遗忘     : decay → archive → consolidate → 用户可见可删
关联     : Task▸Session▸Turn▸Step▸Call▸Artifact 全链 id，一切决策可回溯可归因
```
