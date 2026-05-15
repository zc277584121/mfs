# 客户端与服务端接口设计

本文定义 MFS client 和 MFS server 之间的接口。CLI、Python SDK、多语言 SDK、Skill 都通过同一套 HTTP API 访问 MFS server。server 可以是本机 local daemon，也可以是远程/云端 server。

## 1. 接口边界

稳定接口覆盖这些能力：

- client 可以连接一个 server endpoint：local daemon 或 remote server。
- client 可以**通过 `mfs add` 统一注册并同步本地路径或外部 source**。
- client 可以管理已注册的 source（list/inspect/update/remove）。
- client 可以读取对象和 URI tree。
- client 可以执行 `search/grep`。
- client 可以查看 source、job、MFS Index 和搜索可用性状态。
- `kind=local` profile 处理本机 file path 和 source URI。
- `kind=remote` profile 只处理 source URI；本机 file path 返回错误。

认证与团队能力使用独立接口族：

- 登录命令。
- 工作区切换命令。
- 团队成员和角色。
- 完整认证体系。
- 私有 worker 注册。
- 公开 index 管理 API。

## 2. Client 配置

client 用 profile 指向不同 server。`kind` 字段（不是名字）区分本地/远端：

```toml
[client]
default_profile = "local"

[profiles.local]
api_base_url = "http://127.0.0.1:8765"
kind = "local"

[profiles.prod]
api_base_url = "https://mfs.example.com"
kind = "remote"
credential_ref = "env:MFS_API_TOKEN"
```

`kind = "local"` 表示 client 进程和 daemon 共享同一文件系统命名空间。Docker / SSH / WSL 等跨命名空间场景按 remote 处理。

CLI 请求头：

```text
Authorization: Bearer <token>
MFS-CLI-Version: 0.4.0
MFS-API-Version: v1
```

Profile 管理：

```bash
mfs profile add local --url http://127.0.0.1:8765 --kind local
mfs profile add prod  --url https://mfs.example.com --kind remote
mfs profile use local
mfs profile list
mfs profile status
```

本机 daemon：

```bash
mfs daemon start
mfs daemon status
```

## 3. Add API（统一注册 + 同步入口）

`mfs add` 是统一入口。无论本地路径还是外部 source URI，都走同一个端点；行为幂等。

```text
POST /v1/add
```

请求：

```json
{
  "target": "postgres://prod",
  "config_path": ".mfs/sources/prod-postgres.toml",
  "force": false,
  "since": null,
  "probe": false,
  "watch": false
}
```

`target` 字段同时接受本地绝对路径（如 `/abs/repo`）和 source URI（如 `postgres://prod`）。server 内部按 scheme 路由：

- 没有 scheme 或 scheme=`file` → file source。
- 其他 scheme → 对应 source connector。

响应：

```json
{
  "job_id": "job_01HX...",
  "target": "postgres://prod",
  "status": "queued",
  "is_first_add": true,
  "server_kind": "local",
  "affected_objects": [
    "postgres://prod/public/tickets/rows.jsonl",
    "postgres://prod/public/accounts/rows.jsonl"
  ]
}
```

CLI：

```bash
mfs add .                                                          # 本地
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml   # 外部首次
mfs add postgres://prod                                            # 再同步
mfs add postgres://prod --force                                    # 强制
mfs add slack://eng --since 2026-05-01                             # 增量
mfs add postgres://prod --probe --config ...                       # 试连不写入
mfs add ./repo --watch                                             # 启动 watcher
```

server 负责：

- 校验配置（probe / 首次注册前必校验）。
- 扫描本机文件或拉取外部 source。
- 用 source 的 `sync.py` 做变化检测。
- 构建 normalized content、structure、chunks、summary 和 Retrieval Index。
- 写 metadata、object store 和 search backend。

client 负责：

- 解析 CLI/SDK 参数。
- 把 target 发给 active profile 的 server。
- 输出 job/status/搜索结果。

### 3.1 remote profile 的 file path 响应

remote profile 收到 client 本机 path 时返回：

```json
{
  "error": {
    "code": "remote_server_cannot_read_local_path",
    "message": "Local path requires local profile: ./repo",
    "suggestions": [
      "Run `mfs profile use <local-profile>` and `mfs daemon start`",
      "Use a source URI such as postgres://prod",
      "Use an explicit upload API for cloud indexing"
    ]
  }
}
```

云端文件入口是两个独立能力：

| 能力 | API |
| --- | --- |
| explicit upload | `POST /v1/uploads`（v0.4 不开放，归商业化） |
| server-side file source | `POST /v1/add` + 显式注册 server mounted path |

## 4. Source 管理 API

`/v1/sources` 子树只做**管理**，不再有「add」入口（add 用 `/v1/add` 统一处理）。

```text
GET    /v1/sources
GET    /v1/sources/{source_id}            # 等价 inspect，包含能力声明
PATCH  /v1/sources/{source_id}            # update（替代 source TOML 的局部覆盖）
DELETE /v1/sources/{source_id}
```

CLI：

```bash
mfs source list
mfs source inspect postgres://prod
mfs source update postgres://prod --config .mfs/sources/prod-postgres.toml
mfs source remove postgres://prod
```

`mfs source inspect` 输出包含 source 的能力声明 JSON，agent 据此判断哪些命令对该 source 可用。

## 5. Object API

这些接口对应 `ls/tree/cat/head/tail/export`。API 名用 `objects`，用户文档说 path/object。

```text
GET  /v1/objects/stat?uri=...
GET  /v1/objects/list?uri=...
GET  /v1/objects/tree?uri=...&depth=2
GET  /v1/objects/read?uri=...&range=A:B
GET  /v1/objects/head?uri=...&n=20
GET  /v1/objects/tail?uri=...&n=20
GET  /v1/objects/follow?uri=...           # tail -f 用，SSE 或 chunked stream
POST /v1/objects/export
```

**没有 cursor 参数**。分页用 `range=A:B` 表达；增量数据用 `follow` 端点。

### 5.1 `cat`

CLI：

```bash
mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:2
```

API：

```text
GET /v1/objects/read?uri=postgres%3A%2F%2Fprod%2Fpublic%2Ftickets%2Frows&range=0:2
```

响应：

```json
{
  "source": "postgres://prod/public/tickets/rows.jsonl",
  "media_type": "application/x-ndjson",
  "range": {"start": 0, "end": 2, "total_hint": 12453},
  "items": [
    {"id": 12, "subject": "Login broken after SSO migration"},
    {"id": 41, "subject": "SSO redirect loop"}
  ]
}
```

大对象无 `--range` 的请求返回 `object_too_large_for_cat`。

### 5.2 `tail -f`

```text
GET /v1/objects/follow?uri=slack%3A%2F%2Feng%2Fchannels%2Fincidents%2Ftoday%2Fmessages
Accept: text/event-stream
```

server 用 SSE / chunked HTTP 持续推送新增 record。client 端表现为 `mfs tail -f`。

只有声明 `efficient_tail = true` 的 source 支持 follow，其他返回 `tail_unsupported`。

## 6. Search API

```text
POST /v1/search
POST /v1/grep
```

请求：

```json
{
  "query": "customer cannot login after SSO migration",
  "path": "postgres://prod/public/tickets",
  "top_k": 10,
  "mode": "hybrid"
}
```

响应沿用 CLI JSON envelope：

```json
[
  {
    "source": "postgres://prod/public/tickets/rows.jsonl",
    "lines": null,
    "content": "Login broken after SSO migration",
    "score": 0.842,
    "locator": {
      "kind": "row",
      "primary_key": {"id": 12}
    },
    "metadata": {
      "kind": "search",
      "source_type": "postgres",
      "media_type": "application/x-ndjson"
    }
  }
]
```

## 7. Job 和 Status API

```text
GET  /v1/jobs
GET  /v1/jobs/{job_id}
POST /v1/jobs/{job_id}/cancel
GET  /v1/status?scope=...
```

CLI：

```bash
mfs status
mfs status postgres://prod
mfs status --verbose postgres://prod
mfs status --diagnose
mfs status --watch
mfs job list --failed
mfs job inspect job_01HX...
```

`mfs status` 远程输出示例：

```text
Server: https://mfs.example.com
Sources: 8 active, 1 warning
MFS Index: 24 fresh, 3 stale
Jobs:    2 running, 0 failed
Search:  available
```

`job retry` 不引入：失败时再 `mfs add <uri>` 即可（幂等）。

## 8. 错误格式

```json
{
  "error": {
    "code": "source_unhealthy",
    "message": "Postgres connection failed",
    "source": "postgres://prod",
    "details": {
      "reason": "permission denied for schema public"
    },
    "suggestions": [
      "Check credential_ref secret:postgres-prod-readonly",
      "Grant USAGE on schema public"
    ]
  }
}
```

CLI 文本输出：

```text
Source unhealthy: postgres://prod
reason: permission denied for schema public
try:
  Check credential_ref secret:postgres-prod-readonly
  Grant USAGE on schema public
```

## 9. 版本兼容

- API 使用 `/v1` 主版本。
- CLI 在请求头发送 `MFS-CLI-Version`。
- 服务端在响应头发送 `MFS-API-Version` 和 `MFS-Min-CLI-Version`。
- 字段兼容采用 additive only：添加字段、保持既有字段语义稳定。
- breaking change 进入 `/v2`。

## 10. 认证与组织接口

登录、工作区、团队成员、私有 worker 使用独立接口族；核心 API 只依赖 endpoint、source、object、job、search 这些资源。
