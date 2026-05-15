# 实现、测试与部署计划

## 1. 实现优先级

实现主线是 client/server-first：CLI/SDK 是 HTTP client，本地体验通过 local daemon 跑通。

### 里程碑 1：协议、client 和 local daemon skeleton

目标：

- 定义 `/v1` HTTP API、错误码和 JSON envelope。
- `mfs-cli` 只包含 CLI、Python SDK、HTTP transport、profile 和输出。
- `mfs-server` 提供 API server、local daemon entrypoint、worker entrypoint 和最小 runtime。
- `mfs daemon start` 能启动本机 daemon。
- `mfs profile add/use/list` 能管理 endpoint profile。

验收命令：

```bash
uv tool install mfs-cli
uv tool install mfs-server
mfs daemon start
mfs profile add local --url http://127.0.0.1:8765 --kind local
mfs profile use local
mfs profile status
```

### 里程碑 2：local profile + file source

目标：本地文件体验通过 HTTP 和 local daemon 提供。

```bash
mfs add .
mfs status
mfs search "density preset" . --top-k 5
mfs grep "DensityParams" ./src
mfs tree --peek -L 2 .
mfs ls --skim ./docs
mfs cat --skim ./docs/getting-started.md
mfs cat -n 40:90 ./docs/getting-started.md
```

内部要求：

- CLI 仅负责 HTTP 请求、profile 和输出；目录扫描、hash、chunk 由 daemon 执行。
- CLI 调 `POST /v1/add`。
- local daemon 调 `sources/file/` 读取本机文件。
- `sources/file/sync.py` 使用 manifest 做 `size + mtime_ns + hash-on-change` 对比。
- 文档、代码、表格等对象处理放到 `objects/`。
- embedding、summary、search/grep planning 放到 `pipeline/`。
- metadata、object store、queue、search backend 放到 `storage/`。

### 里程碑 3：source URI 和 fake source

目标：通过 fake source 测试多源行为。

需要 fake source：

- fake GitHub repo/issues/pulls。
- fake Slack。
- fake Postgres。
- fake Zendesk。
- fake Google Drive。

验收命令：

```bash
mfs add postgres://fixture --config tests/fixtures/postgres.toml
mfs ls postgres://fixture
mfs cat postgres://fixture/public/tickets/schema.json
mfs head -n 20 postgres://fixture/public/tickets/sample.jsonl
mfs search "login broken" postgres://fixture/public/tickets
mfs add postgres://fixture   # 幂等再同步
```

### 里程碑 4：结构化对象体验

目标：JSON/JSONL、日志、消息流、大对象区间读取可用。

```bash
mfs head -n 20 postgres://fixture/public/tickets/sample.jsonl
mfs tail -n 10 slack://fixture/channels/incidents/2026-05-10/messages.jsonl
mfs cat postgres://fixture/public/tickets/rows.jsonl --range 0:10
mfs cat postgres://fixture/public/tickets/rows.jsonl --range 100:200
mfs export postgres://fixture/public/tickets/sample.jsonl ./tickets-sample.jsonl
mfs tail -f slack://fixture/channels/incidents/today/messages.jsonl
```

通过标准：

- `cat --range` 在 JSONL 上按 record 切片，**不返回 cursor**。
- 大对象无 `--range` 的 `cat` 请求被拒绝并提示 head/tail/export。
- `tail -f` 用 SSE/chunked stream 推送增量。

### 里程碑 5：remote server 和 Docker

目标：远端 source、后台 worker、Docker 部署跑通。

```bash
docker compose up -d postgres minio redis milvus mfs-api mfs-worker
mfs profile add prod --url https://mfs.example.com --kind remote
mfs profile use prod
mfs profile status
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
mfs job list
mfs search "login broken" postgres://prod
```

remote profile 下的本地 path 错误契约：

```bash
mfs profile use prod
mfs add ./repo
```

应该返回：

```text
Local path requires local profile: ./repo
Run `mfs profile use <local>` and `mfs daemon start` for local files.
```

### 里程碑 6：真实 source

真实 source 接入顺序：

1. GitHub repo。
2. S3 或本地对象存储。
3. Postgres。
4. Slack。
5. Google Drive 或 Feishu Docs。
6. GitHub Issues/PRs、Jira/Linear。
7. Zendesk、Salesforce、HubSpot。

## 2. 端到端测试

### 2.1 Local profile + file path

```bash
mfs daemon start
mfs profile use local
mfs add . --sync
mfs search "queue priority" .
mfs grep "_task_priority" ./src
mfs cat --json ./src/mfs/cli.py
```

通过标准：

- CLI 只通过 HTTP 调 daemon。
- 本地文件搜索、grep、tree、cat 行为与 CLI 契约一致。
- `--json` envelope 稳定。
- `mfs status` 能看到 file source、job、MFS Index 和 search 状态。

### 2.2 Local profile + fake source

```bash
mfs add postgres://fixture --config tests/fixtures/postgres.toml
mfs add slack://fixture --config tests/fixtures/slack.toml
mfs add zendesk://fixture --config tests/fixtures/zendesk.toml

mfs cat postgres://fixture/public/tickets/schema.json
mfs head -n 20 postgres://fixture/public/tickets/sample.jsonl
mfs cat slack://fixture/channels/incidents/2026-05-10/messages.jsonl --range 0:5
mfs search "login broken" zendesk://fixture/tickets
```

通过标准：

- 每个搜索结果都有可继续操作的 URI 和 locator。
- JSONL 对象可以被 `head/tail/cat --range/grep/export` 操作。
- 大对象 `cat` 拒绝并给出 head/tail/export 提示。
- 对象名带 `.jsonl`/`.json` 后缀，跟 unix/Mirage 习惯一致。

### 2.3 Remote profile + source URI

```bash
docker compose up -d postgres minio redis milvus mfs-api mfs-worker
mfs profile use prod
mfs add postgres://fixture --config tests/fixtures/postgres.toml
mfs job list
mfs search "login broken" postgres://fixture
```

通过标准：

- API 创建 job，worker 消费 job。
- metadata、object store、queue、search backend 都写远端部署。
- CLI 输出和 local daemon 场景保持一致。

### 2.4 Remote profile + file path 错误契约

```bash
mfs profile use prod
mfs add ./repo
```

通过标准：

- client 直接拒绝，或 server 防御性拒绝。
- 本地文件内容上传量为 0。
- 错误码是 `remote_server_cannot_read_local_path`。
- 错误信息提示用户切回 local profile。

### 2.5 同步一致性测试

场景：

- 创建对象。
- 修改对象。
- 删除对象。
- 权限变化。
- 游标提交前失败。
- 任务重试（用户手动再 `mfs add <uri>`）。

通过标准：

- 搜索结果随同步变化。
- 删除对象从 search 结果消失。
- 重复 `mfs add <uri>` 保持索引写入幂等。
- 权限变化保持最小可见范围。
- cursor/manifest 只在 job 成功后提交。

## 3. 人类需要准备的环境

智能体可以写代码、fixture、fake server、Docker Compose 和测试脚本，但真实外部 source 需要人类准备。

| Source | 人类准备 |
| --- | --- |
| GitHub | 测试 repo、token 或 GitHub App |
| Google Drive | 测试文件夹、OAuth app 或 service account |
| Feishu | 测试租户、应用权限、文档/群聊 |
| Slack | 测试 workspace、bot token、频道读取权限 |
| Discord | 测试 server、bot token |
| Gmail | OAuth app、测试邮箱或受控账号 |
| Postgres/MySQL | 只读测试库、schema、样例数据 |
| Snowflake/BigQuery | 测试数据集、费用限额、凭据 |
| Salesforce/HubSpot/Zendesk | sandbox、API token、样例对象 |
| 云部署 | 域名、TLS、对象存储、Postgres、队列、向量后端、日志平台 |

## 4. 智能体可以独立完成的工作

- OpenAPI 和错误码草案。
- CLI/SDK skeleton。
- local daemon skeleton。
- TOML 配置解析。
- profile 切换与管理。
- source URI 解析。
- fake source 和 fixture 数据。
- JSON/JSONL `head/tail/cat --range/export`。
- SSE/chunked `tail -f` 实现。
- SQLite metadata。
- 本地 object store。
- queue 接口、本地 SQLite queue、Redis/Postgres queue adapter。
- deterministic embedding 和 fake search backend。
- Docker Compose 本地服务端。
- README、docs、Skill。

## 5. 多语言计划

多语言 SDK 是 client 侧需求；server 内部结构保持稳定。

| 模块 | 交付方式 |
| --- | --- |
| Python CLI/SDK | PyPI `mfs-cli` |
| Python server | PyPI `mfs-server` 和 Docker image |
| TypeScript SDK | npm package，基于 OpenAPI |
| Go SDK | Go module，基于 OpenAPI |
| Java SDK | Maven/Gradle package，基于 OpenAPI |

出现性能压力时，server 内部局部模块按协议边界替换成其他语言实现：

| 候选模块 | 候选语言 | 原因 |
| --- | --- | --- |
| 大目录扫描和 hash | Rust | 性能、跨平台单二进制 |
| 大 JSONL/CSV/Parquet 流处理 | Rust/Go | 内存稳定、流式处理 |
| 高并发 grep/range read | Rust | CPU 和 IO 控制更好 |
| connector SDK glue | Go/TypeScript | 只在目标生态有明显优势时考虑 |

这些改造封装在 server 侧接口后面，CLI、SDK、Skill 和 HTTP API 保持稳定。
