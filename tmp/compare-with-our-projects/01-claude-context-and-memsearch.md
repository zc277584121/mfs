# MFS 与我们自己两个前作的对比

本文记录 MFS（新版重构设计）与我们团队之前做的两个开源项目——**Claude Context** 和 **memsearch**——的对比结论。

目的不是"我们比它们强"的自夸，而是回答两个工程问题：

1. 这两个项目在真实流量下暴露/可能暴露的问题，**哪些是架构层面的根因**（而非孤立 bug）？
2. MFS 这版设计**能不能、以什么方式**把它们重构掉？边界在哪、哪些能力是 MFS 射程之外、必须留在上层？

> 截至文档时间，Claude Context ≈ 11.5k★，memsearch ≈ 1.8k★。结论基于两个仓库最近约两个月的 issue / PR。
> 文中所有 issue 状态都标注了真实情况（已修 / 仍开 / 诊断存疑），**不拿已修复或站不住的 issue 当现存论据**。

---

## 一、一句话结论

> Claude Context 和 memsearch 暴露的架构问题，**几乎都收敛到同一个根因：它们是"薄单进程搜索引擎"，缺一个真正的 server / state 层来仲裁状态、生命周期和并发**。
>
> 而那一层，恰恰是 MFS 相对于"裸搜索引擎"多出来的整个本体（client/server 分离 + DB 任务队列 + fingerprint chain + 处处幂等）。
>
> 所以 MFS 能重构它们的"检索 / 索引 / 状态"那一半，并且是**结构上免疫这类问题**，而不是"逐个修 bug"。但 memsearch 还有"记忆捕获 + 自演化整理 + 知识图谱"半条命，是 MFS 定位之外、必须留在上层 app 的。

一个关键认知：**这些问题跟"搜索质量"正交**。把 hybrid retrieval 做得再好，也解决不了"中途断掉怎么恢复""多进程抢同一个库""清索引清到一半"这些状态/并发问题。

---

## 二、Claude Context

### 2.1 定位

Code search MCP for Claude Code —— 一个 MCP server，把整个 codebase 变成 coding agent 的上下文。早期没太考虑同步/状态，随用户量上来（尤其被 KOL 推荐后的第二波增长）暴露出一批"日常使用才会出现的小设计问题"。

### 2.2 架构根因问题分类

把最近两个月相关 issue/PR 按根因归类，会看到它们是**同几个架构缺陷反复冒头**：

| 根因 | 代表 issue/PR | 本质 |
|---|---|---|
| **A. 没有单一 source of truth**（本地 snapshot / Milvus / 任务状态三方打架） | #295 snapshot 0/0 无限重索引循环、#375 clear_index 要求本地路径存在才删云端、#359 重启后丢失 request 级选项 | 客户端 snapshot 文件被当成真相，和真正的数据（Milvus）脱节 |
| **B. 进程内共享可变状态在多 codebase 间泄漏** | #356 customExtensions 跨 codebase 泄漏、#350 ignorePatterns 跨 codebase 泄漏（#357/#351 已修） | 长生命周期 `Context` 实例被多项目复用，直接 mutate 全局字段 |
| **C. 多进程/多 session 重复扫描、并发冲突** | #347 防止并发 background sync、#314 background sync 改 opt-in | 每个 editor session 一个 MCP 进程，各扫各的，互相踩 |
| **D. 索引生命周期不清晰、半截状态无法恢复** | #295（kill 在 reindex 中途）、#354 清理孤儿 merkle snapshot | "索引到一半被打断 / 被 clear" 没有定义好的状态机 |
| **E. 搜索结果重复** | #333 去重重叠 chunk | AST splitter overlap 窗口导致同段代码被召回 2–3 次 |
| **F. 多项目共享 / 全局知识库缺失** | #374 全局知识库、#373 用 git remote 命名团队共享 collection | 单项目 scope，无法跨项目复用索引 |
| **G. 配置全局单例，无法 per-repo** | #228/#308 换 embedding 模型要"删光索引+删 snapshot+改 .env+全量重建" | embedding provider/model/维度全局共享，任意时刻只能一个 |

A/C/D 尤其典型——不是"少写个 if"，是当初把状态分散在"客户端 snapshot + 多个 MCP 进程内存 + Milvus"、且没有统一仲裁者导致的结构性病。

### 2.3 新设计如何 cover

| 根因 | MFS 怎么治 | 设计出处 |
|---|---|---|
| A source of truth | 上游/文件是唯一真相；metadata DB 只是"认知"，Milvus/object store 是派生。client 几乎零持久状态，连 file manifest 都在 server 的 `file_state` 表 → #295 那种"客户端 snapshot 当真相"的 bug 类不存在 | 01 §4、02 §3.4、02 §7 |
| B 状态泄漏 | 没有被多项目复用的进程内可变状态；config 是 DB 里按 connector 隔离的数据；chunk_id 焊死 `namespace_id + connector_uri` → 两项目天然不串台 | 02 §2、02 §7① |
| C 并发 | 薄 client + 单一重 server，状态集中一处；并发进 `connector_jobs` 表 + unique 约束（同 connector 至多一个 running + 一个 queued）；v0.4 直接砍掉内置定时调度器，sync 由用户显式触发（最强版的"explicit and cheap"） | 01 §1、02 §8.1 |
| D 生命周期/恢复 | per-object 原子 + state 末尾提交 + remove 抢占 sync（跑完手头 object 再停）；恢复模型坍缩成"重跑 = 下次 mfs add"；chunk_id 幂等保证重跑不脏 | 02 §7、§8.3 |
| E 结果重复 | `mfs search --collapse object` 按 object 折叠（粒度与 #333 的"按行重叠合并"略不同，覆盖"不该重复"诉求） | 06 §search |
| F 跨项目 | 查询侧已覆盖：`mfs search --all` 跨 connector 检索 | 06 §9 |
| G per-repo 配置 | v0.5+ config-profile：每组打包一套 converter+embedding+VLM+summary，对应一张自己的 collection（只有 embedding 模型决定 collection 切分）；connector 归属一个组，搜索按组各出一份、不强行跨向量空间合并 | 06 §config-profile slot |

**元层面**：A/B/C/D 不是被"分别打 4 个补丁"，而是被**同一个架构决策**一次性消掉——把状态从"分散无仲裁"收敛到"单一 server 的 metadata DB + client 无状态 + 一切幂等"。这些 bug 在新模型里**根本表达不出来**。

### 2.4 v0.4 主动不做 / 推迟 v0.5+

不夸大，几条诉求 v0.4 是**故意没做、文档明确标注的范围切割**：

| 缺口 | 对应 | 现状 |
|---|---|---|
| 同一份文件不同 client 环境 = 不同 connector（Docker/CI/WSL 重复索引） | #373 | v0.4 已知限制，缓解靠共享 `~/.mfs`；彻底解耦（`--alias`）+ 多 client 并发写协调留 v0.5+（02 §4.2） |
| 全局知识库共享同一索引身份 | #374 | 查询侧已覆盖；"索引一次多项目共享身份"依赖 alias，v0.5+ |
| 换模型/chunker 后自动失效重建 | — | v0.4 手动 `mfs add --force-index`；自动检测配置漂移 v0.5+（04 §5.2） |
| per-repo embedding | #228 | v0.5+ config-profile |
| 多 client 并发写同一 connector | — | v0.4 禁止（拒绝），真正协调 v0.5+ |

### 2.5 Claude Context 能、MFS 不能（且我们判断不重要）

诚实列出，但都不构成重要短板：

| 能力 | 性质与判断 |
|---|---|
| **MCP 原生集成** | Claude Context 本身是 MCP server，插进 Cursor/Claude Desktop/VSCode/Zed。MFS 走 CLI + Skill。**但我们的判断：Skill > MCP 是当前风向**——Skill 能教 agent 玩一整套 CLI 的组合编排（`tree --peek`→`cat --skim`→`search`、何时 grep 何时 search、`--range` 验证），这是 MCP"一动作一 tool"难表达的；CLI 还能自由拼管道。MCP 反而更重（限制表达力 + 维护 schema）。**此项不补。** |
| 编辑器 GUI（VSCode 扩展 / webview / Zed） | MFS 定位 agent + 终端，主动放弃 GUI 面 |
| 零服务单进程上手（`npx` 即用） | MFS 是 C/S，启动摩擦更高；这是换健壮性/一致性的合理代价（`mfs serve`/autostart 缓解） |
| 向量库可插拔（Qdrant，ES 在请求中） | MFS 战略锁定 Milvus（schema 焊死）；接受"放弃非 Milvus 用户" |
| 浏览器内纯客户端（IndexedDB 向量库） | 定位之外 |

---

## 三、memsearch

### 3.1 定位

基于 Markdown + Milvus 的统一记忆层（Claude Code / Codex 等 agent 的持久记忆）。内核：**Markdown=真相源 → Engine 切块/embed → Milvus=可重建影子索引**；CLI 为 `index / search / watch / compact`；SHA-256 去重；三层渐进检索（L1 chunk → L2 section → L3 原始 transcript）；通过 hook plugins 捕获对话、LLM 总结、追加写每日 `.md`。

### 3.2 它比 Claude Context 健康在哪

**memsearch 从一开始就把 source of truth 立对了**：markdown=truth、Milvus 可重建。所以它**没有** Claude Context 那个要命的 A 类病（#295 snapshot 当真相→无限重索引）。中途断掉后，因为 markdown 是真相，重跑能自然 re-sync（只要不踩下面 #533 的部分输入误删）。

这就是它流量更小之外、还更稳的根本——**架构起点正确**。它现在的 issue 大多是边角和 plugin 打磨，而不是架构性大坑，印证了这点。

### 3.3 它仍然有的架构问题（用 Claude Context 的类别去套）

memsearch 跟 Claude Context 同样是"薄单进程搜索引擎"，所以撞同一类墙——只是撞得轻：

| memsearch issue | 状态 | 对应 Claude Context 类别 | 本质 |
|---|---|---|---|
| **#80 `watch` 锁住 milvus.db，导致 `search` 直接打不开** | CLOSED | **C 并发**（比 #347 更狠：不是浪费，是 search 整个挂） | Milvus Lite 单文件、不支持多进程并发；watch 进程独占锁，另一进程的 search 连库都开不了 |
| **#352 watcher 回调的 try/except 被删 → 单个文件出错崩掉整个 watcher** | CLOSED | **D 中途断掉/恢复韧性** | 单点失败拖垮整体、不可恢复 |
| **#533 OOM mid-corpus 杀进程；且 `delete_by_source` 把"不在本次入参的源"当删除清掉** | OPEN（窄场景，报告人已无法复现） | **D 中断 + deletion 误推断** | 部分/分批输入会互相把对方当删除 |
| #203 Lite 模式 index 无限 hang | CLOSED | D liveness | 无超时/恢复 |
| 全局 embedding 配置（#88/#194/#299 都是"加 provider"） | — | **G 配置全局单例** | 一次只能一个模型；**但 memory 场景固定，几乎不触发**——不是真痛点 |

> **诚实边界（不当现存论据用）**：以下我曾提及但站不住或已修——#324/#199 子目录用错 collection（已修）、#539 batch 内重复主键（已修）、#526 超长行（已修 PR #528）、#540/#542 collection released（已修 PR #544）、#534 upsert 没 flush（**诊断存疑**：维护者指出 remote Milvus 默认 Bounded 一致性，stats 滞后不证明丢写）、#538 config-only 显示 0 chunks（零跟进的小边角）、#520/#527（plugin 层，处理中）。

### 3.4 统一根因 → MFS 为什么结构上免疫

#80 / #352 / #533 / #203 不是孤立 bug，是"**memsearch 没有 server/state 层**"这一个根因的不同投影：

| memsearch 的病 | 根因（同一个） | MFS 为什么结构上长不出来 |
|---|---|---|
| #80 watch 锁库、search 挂 | 多进程各自直连嵌入式 Milvus | **单一 server 独占持有 Milvus 连接**，所有 client 走 HTTP；多个 CLI 并发都不碰 .db 文件，锁冲突不存在（02 §1） |
| #352 watcher 单点崩、#203 hang | 无 worker 隔离 + 超时恢复 | server 侧 worker pool + 幂等 + 心跳超时重置 + circuit breaker；单个坏 object 标 failed，不拖垮整体（02 §7.1） |
| #533 部分输入→误删 + OOM 中断 | 朴素 delete_by_source + 无续跑 | deletion 分模式：incremental 不推断删除、只有 full_scan 全集 diff 才删；中途崩 state 不 commit，下次 `mfs add` 续跑（02 §7.4） |
| 全局 embedding | 配置全局单例 | v0.5 config-profile（memory 场景非必需） |

**准确说法**：不是"memsearch 现在 bug 多、MFS 能治"，而是"**memsearch 结构上注定会反复长出这类病，MFS 从根上就长不出来**"。

### 3.5 memsearch 有、MFS 没有的能力（重构边界）

这是重构时必须明确的边界——这些是 feature/scope，不是 bug，**MFS 给不了，要留在 memsearch 自己那层**：

| memsearch 能力 | 是什么 | MFS 现状 |
|---|---|---|
| **① 记忆写入/捕获** | 对话 → LLM 总结 → 带 session anchor 追加进每日 `.md` | **MFS connector 契约只读**（list/stat/read/fingerprint/sync），**没有 write/append object 动词**。最大 gap |
| **② compact** | 把已存 chunk 做一次性 LLM 压缩总结 | MFS 有 summary 派生 artifact，但不"改写/压缩源文件" |
| **③ dreaming / 记忆整理（#523）** | 后台周期 review、去重、凝练旧记忆；按 topic/entity 建知识 wiki；防文档腐烂 | 完全没有；MFS 是读侧、不 mutate 源 |
| **④ authored edges / 图（#407）** | 文档间带谓词的边（`has_form_contract::[[X]]`）+ 图遍历 | MFS 只有向量+BM25，无图/边模型 |
| **⑤ L3 turn-anchor 召回（#505/#506）** | 用 session/turn 锚点精确回放原始 transcript | MFS 有 locator+chunk_kind，部分可表达，但 turn 锚点的写侧 schema 是 memsearch 特有 |

> 注意：plugin/hook 层的一批脆弱性（#520 stop hook 递归触发 SessionStart、#527 限流错误字符串写成记忆、#546 Windows unicode、#521 SIGPIPE、路径空格等）都属于**捕获管线**——MFS 既不"有"也不"治"，因为它不做捕获。重构后这层仍由 memsearch 维护。

### 3.6 重构关系

> **memsearch 退化成 MFS 之上的一个记忆 app**——保留捕获 plugins（hook 写 `.md`）+ compact + dreaming + 图，把内部的 index/search/store/状态 全部换成调 MFS。

这恰好就是 MFS README 那张图里 "memory systems (daily .md logs)" 那个 Agent Application 方框。换进去后：

- ✅ §3.3/§3.4 的并发、中断、误删、身份问题在结构上消失；
- ❌ 捕获层脆弱性仍在（属于 memsearch 自己那层）；
- ❌ ①③④能力 MFS 给不了，memsearch 必须自己在 MFS 之上实现。

---

## 四、总结：两个项目共同教给我们的

1. **真正的难点不是大功能，是"跑起来每天用"才暴露的小设计**：同步、source of truth、半截被打断、多 session 并发。这些 KOL demo 看不出来，真实流量一上来全冒头。
2. **这类问题跟 search 质量正交**——它们是 state / lifecycle / concurrency 问题。Claude Context 和 memsearch 都缺一个 server/state 层，所以都撞；memsearch 因为 truth 模型干净 + 场景固定，撞得轻。
3. **MFS 的本体价值就是补上这层**：client/server 分离把仲裁者立起来、DB 任务队列管生命周期、fingerprint chain 管增量、处处幂等让恢复坍缩成"重跑"。这让上述整类问题**结构上免疫**，而不是逐个修。
4. **重构关系清晰**：
   - **Claude Context** → 检索/索引/状态层可整体被 MFS 重构（且更稳）；MCP 接口我们不补（Skill 取代）。
   - **memsearch** → 检索/索引/状态层可被 MFS 取代；但"记忆捕获 + 自演化整理 + 知识图谱"留在上层 app，memsearch 成为 MFS 之上的记忆系统。
5. **诚实纪律**：对比时只用站得住的现存问题论证，已修/诊断存疑的（§3.3 那批）明确剔除，避免"拿修好的 bug 证明自己强"。

---

## 附录：外部同类项目定位速览

为补全竞品地图，附上之前讨论的四个外部项目（详细对比另见，此处只记定位差）：

| 项目 | 定位 | 与 MFS 的核心差异 |
|---|---|---|
| **OpenViking**（字节） | AI Agent 的上下文数据库（memory/resource/skill），会自演化 | 最像；但它是"agent 大脑/会变的活上下文"，MFS 是"文件为真相、Milvus 为派生"的克制搜索层 |
| **Mirage**（Strukto） | 给 agent 的统一虚拟文件系统（真 FUSE 挂载） | Mirage 解决"怎么访问"且不做检索；MFS 解决"怎么搜到"，不挂载 |
| **CocoIndex** | 增量数据 ETL 框架 | 是 MFS 服务端 ingest 层的同类；没有 agent 浏览/搜索前端 |
| **LlamaCloud**（LlamaIndex） | 托管的文档解析+索引+检索 SaaS | 卖解析质量+托管给开发者；MFS 开源自托管、给 agent 的 shell 自助检索 |

MFS 的独特组合：**shell-native CLI（agent 第一接口）+ 一等公民混合检索 + 薄客户端/重服务端 + 文件为真相 + Airbyte 式社区 connector 生态**——四样齐备的只有 MFS。
