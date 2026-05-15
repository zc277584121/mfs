# MFS 设计总览

MFS 是 **Multi-source File-like Search**：让 agent 用一套 shell-native CLI 搜索、浏览和读取本地文件、代码仓库、云文档、消息、ticket、数据库和 SaaS 记录。所有数据通过路径或 URI 寻址，所有命令是 POSIX 风格（`ls / cat / grep / head / tail`）；**不挂载文件系统**。

`mfs add` 是统一注册 + 同步入口（幂等），本地路径和外部 source URI 共用。`mfs source` 子树只做管理（list/inspect/update/remove）。日常使用 `mfs add/search/grep/ls/tree/cat/head/tail/export/status`。

实现方向收敛为 client/server-first：CLI、SDK、Skill 都是 client；source/connector、同步、MFS Index、embedding、Retrieval Index 和存储都在 server 侧执行。本地体验通过本机 local daemon 实现，远端体验通过 remote server 实现；profile 的 `kind` 字段（local / remote）决定行为。

一句话：

> MFS lets agents search and inspect local files, repos, cloud docs, chats, tickets, databases, and SaaS records through one shell-native CLI. No mount, no SDK — just a binary on $PATH.

## 阅读顺序

1. [01-cli-behavior-and-compatibility.md](01-cli-behavior-and-compatibility.md)：CLI 行为、路径规则、多源命令和 JSON 输出。
2. [02-local-file-and-repo-scenarios.md](02-local-file-and-repo-scenarios.md)：本地目录、代码仓库、GitHub repo 的用户流程。
3. [03-cloud-file-source-scenarios.md](03-cloud-file-source-scenarios.md)：Google Drive、Feishu Docs、S3 等文件型 source。
4. [04-message-source-scenarios.md](04-message-source-scenarios.md)：Slack、Discord、Gmail、飞书群聊等消息型 source。
5. [05-task-and-collaboration-source-scenarios.md](05-task-and-collaboration-source-scenarios.md)：Jira、Linear、GitHub Issues/PRs、Notion database。
6. [06-database-source-scenarios.md](06-database-source-scenarios.md)：Postgres、MySQL、MongoDB、Snowflake、BigQuery。
7. [07-business-saas-source-scenarios.md](07-business-saas-source-scenarios.md)：Salesforce、HubSpot、Zendesk。
8. [08-output-paths-and-json-contract.md](08-output-paths-and-json-contract.md)：本地路径、外部 URI、JSON 输出、分页与错误格式。
9. [09-config-readme-and-skill-plan.md](09-config-readme-and-skill-plan.md)：配置、README、docs、Skill 应该怎么讲。
10. [10-implementation-test-and-deployment-plan.md](10-implementation-test-and-deployment-plan.md)：实现优先级、测试、上云和人工依赖。
11. [11-architecture-design.md](11-architecture-design.md)：从用户命令到资源树、索引、同步和权限的整体架构。
12. [12-client-server-api-design.md](12-client-server-api-design.md)：客户端与服务端接口、请求响应、错误和版本兼容。
13. [13-cloud-business-and-deployment.md](13-cloud-business-and-deployment.md)：业务上云、团队空间、计费、部署和运维。
14. [14-project-structure-watch-and-release.md](14-project-structure-watch-and-release.md)：项目目录结构、worker/queue/sync 边界、目录监控权限、打包发布和版本策略。
15. [15-command-inventory-and-scope.md](15-command-inventory-and-scope.md)：公开命令清单（16 个顶级命令）和职责边界。
16. [16-concepts-paths-anchors-and-command-decisions.md](16-concepts-paths-anchors-and-command-decisions.md)：URI、本地路径、locator、Source/Object、分页、密度视图等概念决策。
17. [17-source-sync-and-mfs-index.md](17-source-sync-and-mfs-index.md)：各类 source 的同步策略、MFS Index、URI 映射和 Retrieval Index。
18. [18-project-structure-flow.html](18-project-structure-flow.html)：可交互目录树，点击查看 local daemon/remote server 与 file/source 四种信息流，以及 PyPI/Docker 交付边界。

## 最重要的产品原则

### 1. 本地文件入口简单直接

本地文件使用普通路径：

```bash
mfs daemon start
mfs profile use local
mfs add .
mfs status
mfs search "how do we handle token expiration" .
mfs grep "ERR_TOKEN_EXPIRED" .
mfs tree --peek -L 2 ./docs
mfs cat --skim ./docs/auth.md
mfs cat -n 40:90 ./docs/auth.md
```

这些命令通过 HTTP 调本机 local daemon；daemon 负责 file source、对象处理、MFS Index 和 Retrieval Index。用户只需要理解本地 path 和 MFS 命令。

### 2. 外部 source 使用同一套命令

外部 source 使用 URI，本地文件继续使用普通路径，**同一个 `mfs add` 入口幂等处理两者**：

```bash
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml   # 首次：注册+同步
mfs add postgres://prod                                            # 再同步一次
mfs add postgres://prod --force                                    # 强制重建
mfs search "customer cannot login" postgres://prod/public/tickets
mfs grep "timeout" slack://eng/channels/incidents
mfs ls zendesk://support/tickets
mfs cat github://mfs/README.md
```

日常使用仍然是 `add/search/grep/ls/tree/cat/head/tail/export`。`mfs source` 子树只管 list/inspect/update/remove。

### 3. Agent 看到的是可操作结果

搜索结果返回可继续操作的 path/URI：

```text
[1] postgres://prod/public/tickets/rows.jsonl  score=0.842
    row: id=12
    subject: Login broken after SSO migration
    status: open
    priority: high
```

Agent 可以继续读取或过滤：

```bash
mfs grep '"id":12' postgres://prod/public/tickets/rows.jsonl
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:100
mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
```

### 4. 对象名带 media type 后缀

每个 source 决定自己 root 下面暴露哪些对象。对象名带 media type 后缀（`schema.json` / `sample.jsonl` / `rows.jsonl` / `messages.jsonl` / `records.jsonl` / `database.json` 等），跟 unix 习惯和 Mirage 一致。

```bash
mfs ls postgres://prod/public/tickets
# schema.json
# sample.jsonl
# rows.jsonl

mfs cat postgres://prod/public/tickets/schema.json          # JSON 渲染
mfs head -n 5 postgres://prod/public/tickets/sample.jsonl    # JSONL 渲染
```

每个 source 暴露哪些对象是稳定契约，写在 source 文档和 `mfs source inspect` 输出里。「原对象 vs 索引产物」对用户透明——所有对象都只是「source 下面的对象」，cat 它就行。

### 5. 分页用 range，不用 cursor

```bash
mfs head -n 20 <uri>          # 前 N 行/记录
mfs tail -n 20 <uri>          # 后 N 行/记录
mfs tail -f <uri>             # 流式跟随
mfs cat <uri>                 # 完整对象；大对象拒绝并提示
mfs cat <uri> --range A:B     # 区间读取
mfs export <uri> <file>       # 完整导出到本地
```

**没有 cursor token**。需要遍历大对象时用 `mfs export` 物化到本地后处理。

### 6. 架构先服务用户行为

内部需要资源树、缓存、索引、权限和同步。文档主线先回答：

- 用户怎么添加 source？
- 用户怎么浏览？
- 用户怎么搜索？
- 命令输出是什么？
- 大对象怎么读？
- 搜索结果怎么打开？
- 哪些操作需要确认？
- 出错时用户如何修复？

## 核心命令分层

| 命令 | 用途 | 说明 |
| --- | --- | --- |
| `mfs daemon start` | 本机运行时 | 启动本机 local daemon |
| `mfs profile add/use` | 连接选择 | 用 `kind=local` 或 `kind=remote` 注册并选择 endpoint |
| `mfs add <path-or-uri>` | 统一注册+同步 | 幂等处理本地路径和 source URI |
| `mfs status` | 状态查看 | 显示 daemon/profile、source、sync job、MFS Index 和搜索可用性 |
| `mfs source list/inspect/update/remove` | Source 管理 | 不再有 `source add`（合并到 `mfs add`） |
| `mfs search <query> <path>` | 语义搜索 | 对本地路径或 source URI 搜索 |
| `mfs grep <pattern> <path>` | 精确搜索 | source-aware exact search，可下推 |
| `mfs ls/tree <path>` | 浏览结构 | 浏览本地路径或 source URI |
| `mfs cat <path>` | 读取对象 | 读文件或外部对象，支持 `--range A:B`；大对象拒绝 |
| `mfs head/tail` | 行/记录预览 | `tail -f` 跟随 append-only 对象 |
| `mfs export <path> <file>` | 导出 | 把对象写到本地文件 |
| `mfs job list/inspect/cancel` | 后台任务 | 看 sync/index/export job |

共 **16 个顶级命令**（11 动词 + 5 名词），详见 [15-command-inventory-and-scope.md](15-command-inventory-and-scope.md#12-数量总结)。

## 最小心智模型

```text
mfs daemon start                 启动本机 local daemon
mfs profile use local            选择本机 daemon
mfs add .                        给工作目录建索引（注册+同步，幂等）
mfs add <source-root> --config   首次注册一个外部 source
mfs add <source-root>            已注册：再同步一次
mfs tree <path-or-uri>           看 MFS 能访问什么
mfs search "..." <path>          找相关内容
mfs grep "..." <path>            找精确字符串
mfs cat <result-path>            打开搜索结果
mfs cat <uri> --range A:B        按区间读大对象
mfs tail -f <append-log-path>    跟随日志或消息流尾部
mfs export <uri> <file>          把大对象写到本地后处理
```

## 三层概念命名

- **MFS Index**（用户面总称）：一个 source 在 MFS 内的全部索引集合，包含两个子层：
  - **Index Artifacts**：metadata、normalized content、structure、chunks、summary、JSONL page cache 等可 `cat / head / tail` 的产物。
  - **Retrieval Index**：向量 + BM25，给 `search / grep` 用。
- **Index Plan**（开发者面）：source TOML 里描述「该生成哪些 Artifacts」的配置。
