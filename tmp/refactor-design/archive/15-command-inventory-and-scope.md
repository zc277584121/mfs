# 命令清单与职责边界

本文回答公开命令有哪些、每类命令负责什么。命令设计按 client/server-first 收敛：CLI 是 client，所有重活都通过 HTTP 交给 local daemon 或 remote server。

## 1. 设计原则

命令清单遵循三条原则：

- **动词命令 POSIX 风格**：`ls / cat / grep / head / tail / ...` 跟系统工具同名，agent 不用记新词。
- **名词命令 noun-verb 风格**：`source list`、`profile add`、`daemon start` 等，所有管理类操作集中在名词子树下。
- **入口尽量少**：能用一个命令的幂等行为表达，就不引入第二个命令。

## 2. 顶级命令一览

公开的顶级命令分两类。

### 2.1 动词命令（POSIX 风格）

| 命令 | 作用 |
| --- | --- |
| `mfs add <path-or-uri>` | 注册并同步本地路径或外部 source。**幂等**：再跑一次等于「再同步一次」 |
| `mfs status [<path-or-uri>]` | 查看 daemon/profile、source、MFS Index、search 可用性、job 状态 |
| `mfs search <query> <path-or-uri>` | 语义/关键词混合搜索 |
| `mfs grep <pattern> <path-or-uri>` | 精确搜索，能下推 source 时下推 |
| `mfs ls <path-or-uri>` | 浏览目录或 source 暴露的对象 |
| `mfs tree <path-or-uri>` | 树状浏览 |
| `mfs cat <path-or-uri>` | 读取对象；大对象直接拒绝并提示 head/tail/export |
| `mfs head <path-or-uri>` | 看前 N 行/记录 |
| `mfs tail <path-or-uri>` | 看后 N 行/记录；`-f` 流式跟随 append-only source |
| `mfs export <path-or-uri> <file>` | 把对象写到本地文件 |
| `mfs remove <path-or-uri>` | 从 MFS Index 移除 |

`config` 也保留为动词常用入口，详见 2.2。

### 2.2 名词命令（noun 子树）

| 命令 | 作用 |
| --- | --- |
| `mfs source list/inspect/update/remove` | 已注册 source 的管理 |
| `mfs profile add/use/list` | client endpoint profile（替代旧设计里的 `mfs server`） |
| `mfs daemon start/stop/status/logs` | 本机 server 进程管理 |
| `mfs job list/inspect/cancel` | 后台任务可见性 |
| `mfs config show/set` | 查看与修改配置 |

合计顶级命令 **16 个**：11 个动词命令 + 5 个名词命令。

## 3. 与之前设计的差异

| 旧设计 | 新设计 | 原因 |
| --- | --- | --- |
| `mfs source add <uri>` | 合并进 `mfs add <uri>` | 本地 path 和外部 source 共享一个幂等入口 |
| `mfs sync <uri>` | 合并进 `mfs add <uri>` | 再 add 一次等于同步；不引入第二个动词 |
| `mfs source validate` | 合并进 `mfs add`（注册前自动校验） | 校验是 add 的前置步骤，不需要独立命令 |
| `mfs source healthcheck` | 合并进 `mfs status <uri>` | 健康本来就是 status 的一部分 |
| `mfs server use/list/status` | 改名 `mfs profile use/list/add` | 「server」一词与 server 进程概念重复，profile 表达更准 |
| `mfs watch list/stop/doctor` | 删除 | `add --watch` 启动；`status --watch` 看；不需要独立子树 |
| `mfs doctor` | 改 `mfs status --diagnose` | 不值得一个顶级命令 |
| `mfs stat` | 删除 | 跟 `mfs ls --json <single-uri>` 重叠 |
| `mfs jq` | 删除（暂不引入） | 想做 server-side pushdown 时再加；纯 client-side 用本地 jq 即可 |
| `mfs upload` | 移出 v0.4，归商业化能力 | 跟 `mfs add` 语义冲突；云端上传是另一条产品线 |
| `mfs login/workspace/user/org/worker` | 移出 v0.4，归认证/商业化 | 不属于核心 |

## 4. `mfs add` 是统一入口

本地路径和外部 source URI 共用同一个动词。`mfs add` 的行为是**幂等的**：

```bash
# 第一次跑：注册 + 同步
mfs add ./repo
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml

# 已注册：等价于再同步一次
mfs add ./repo
mfs add postgres://prod

# 强制重建
mfs add ./repo --force
mfs add postgres://prod --force

# 增量同步从某个时间点
mfs add slack://eng --since 2026-05-01

# 试连不写入
mfs add postgres://prod --probe --config .mfs/sources/prod-postgres.toml
```

- `--config <toml>` 只在首次注册时需要；已注册时忽略（或 `--config` 配合 `--update` 更新配置）。
- `--force` 跳过快速判断，重建可重建的 Index Artifacts。
- `--since <date>` 给支持游标/时间的 source 用。
- `--probe` 试连接、试校验，不写状态。

source TOML 改动后想更新已注册的 source：

```bash
mfs source update postgres://prod --config .mfs/sources/prod-postgres.toml
mfs add postgres://prod   # 用新配置再同步一次
```

## 5. Runtime：daemon 与 profile

### 5.1 `mfs daemon`

`mfs daemon` 管理本机 server 进程。它跟 profile 概念是正交的——`profile` 决定 client 连哪个 endpoint，`daemon` 决定本机是否跑 server 进程。

| 命令 | 作用 |
| --- | --- |
| `mfs daemon start` | 启动本机 local daemon |
| `mfs daemon stop` | 停止 |
| `mfs daemon status` | PID、端口、版本、健康 |
| `mfs daemon logs` | 日志位置或近期日志 |

如果只装了 `mfs-cli` 没装 `mfs-server`：

```text
Local daemon requires mfs-server.
Install it with:
  uv tool install mfs-server
```

### 5.2 `mfs profile`

`mfs profile` 管理 client active endpoint。

| 命令 | 作用 |
| --- | --- |
| `mfs profile add <name> --url <url> --kind local\|remote` | 注册一个 endpoint profile |
| `mfs profile use <name>` | 切换 active profile |
| `mfs profile list` | 列出所有 profile |
| `mfs profile status` | 检查 active profile 的可达性、版本和 API 兼容性 |

`kind` 字段决定行为，**不靠名字**：

- `kind = local`：client 进程和 daemon 共享同一文件系统命名空间。本地路径请求由 daemon 直接读文件。
- `kind = remote`：跨网络。本地路径请求返回 `remote_server_cannot_read_local_path`。

`mfs login`、team、BYOC、私有 worker 注册属于认证与组织管理接口族，不在 v0.4 范围。

## 6. Source 管理

`mfs source` 子树只保留**管理类**子命令。注册与同步交给 `mfs add`。

| 命令 | 作用 |
| --- | --- |
| `mfs source list` | 列出已注册 source |
| `mfs source inspect <root-uri>` | 看配置、能力、状态、暴露的对象 |
| `mfs source update <root-uri> --config <toml>` | 更新 source 配置 |
| `mfs source remove <root-uri>` | 删除 source |

示例：

```bash
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
mfs source list
mfs source inspect postgres://prod
mfs source update postgres://prod --config .mfs/sources/prod-postgres.toml
mfs add postgres://prod
```

`mfs source inspect` 输出包含 source 的[能力声明](17-source-sync-and-mfs-index.md#8-source-能力声明)，agent 可以据此判断哪些命令对该 source 可用。

## 7. 状态与同步

`mfs status` 是统一的状态入口。MFS Index 同步进度、source 健康、search 可用性、job 状态都从这里看。

| 用法 | 作用 |
| --- | --- |
| `mfs status` | 总览：active profile、daemon、所有 source、jobs |
| `mfs status <path-or-uri>` | 单个 source/路径的详细状态 |
| `mfs status --verbose <uri>` | 加上 retrieval index、freshness、artifact 版本 |
| `mfs status --diagnose` | 自检 daemon、profile、source、storage、search backend |
| `mfs status --watch` | 看正在运行的 watcher |

样例：

```text
$ mfs status
Profile: local (kind=local)
Daemon:  running (pid=4112, port=8765, version=0.4.0)
Sources: 3 active
  ./repo                    last_add=2026-05-14T09:21:00Z   index=fresh
  postgres://prod           last_add=2026-05-14T07:00:00Z   index=stale
  slack://eng               last_add=2026-05-13T22:00:00Z   index=fresh
Jobs:    1 running, 0 failed
Search:  available
```

## 8. 对象读取与遍历

`ls/tree/cat/head/tail/export` 覆盖了浏览到读取的全谱：

| 命令 | 用途 |
| --- | --- |
| `mfs ls <uri>` | 列对象 |
| `mfs tree <uri>` | 树状浏览 |
| `mfs cat <uri>` | 完整读取；大对象拒绝并提示 |
| `mfs cat <uri> --range A:B` | 按行/记录区间读取 |
| `mfs head -n N <uri>` | 前 N 行/记录 |
| `mfs tail -n N <uri>` | 后 N 行/记录 |
| `mfs tail -f <uri>` | 流式跟随（append-only） |
| `mfs export <uri> <local-file>` | 写本地文件 |

**没有 cursor token**：分页用 `--range` 表达，不可重入的"翻页 token" 不暴露给用户。详细分页设计见 [16-concepts-paths-anchors-and-command-decisions.md](16-concepts-paths-anchors-and-command-decisions.md#7-分页设计)。

密度视图：

```bash
mfs tree --peek -L 2 ./docs/
mfs ls --skim ./docs/
mfs cat --skim ./docs/auth.md
mfs cat --deep ./docs/auth.md
mfs cat -n 40:60 ./docs/auth.md
```

`--peek/--skim/--deep` 主要面向 document/code 形态的对象；对结构化 source 的退化行为见 [16](16-concepts-paths-anchors-and-command-decisions.md#8-密度视图与-headtail-的分工)。

## 9. Job 命令

local daemon 和 remote server 都有后台 job。

| 命令 | 作用 |
| --- | --- |
| `mfs job list [--failed]` | 查看任务 |
| `mfs job inspect <id>` | 任务详情 |
| `mfs job cancel <id>` | 取消任务 |

`job retry` 不引入：失败时再 `mfs add <uri>` 即可（幂等）。

## 10. Watch

watch 不再有独立子树。

| 用法 | 作用 |
| --- | --- |
| `mfs add <path> --watch` | 启动 watcher |
| `mfs add <path> --no-watch` | 显式禁用（覆盖配置默认） |
| `mfs status --watch` | 看哪些路径在 watch |
| `mfs remove <path>` | 删除索引同时停 watcher |

## 11. 商业化与认证

这些命令不在 v0.4 范围：

| 命令 | 归属 |
| --- | --- |
| `mfs login` | 认证产品 |
| `mfs workspace ...` | 多 workspace、团队 |
| `mfs org ...` / `mfs user ...` | 组织管理 |
| `mfs worker register` | 私有 worker / BYOC |
| `mfs upload ...` | 云端文件上传，独立审计与权限 |

它们之后以独立接口族出现，URL/命令风格保持一致即可。

## 12. 数量总结

顶级命令（v0.4 范围）：

- 动词类：`add` `search` `grep` `ls` `tree` `cat` `head` `tail` `export` `status` `remove` —— 11 个
- 名词类：`source` `profile` `daemon` `job` `config` —— 5 个

合计 **16 个**。与之前文档「9 个 + 3 个 = 12 个」的数字不一致，本次以本表为准。
