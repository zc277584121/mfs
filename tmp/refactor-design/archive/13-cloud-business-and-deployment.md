# 业务上云与部署设计

本文描述上云设计。remote MFS server 负责运行远程 source、后台同步、MFS Index 更新、远程搜索和对象读取。团队、认证、私有 worker、复杂权限和计费套餐属于独立商业化能力。

远端 server 处理 source URI。本机普通路径由 local daemon 处理；remote profile 下的 `mfs add ./repo` 返回 `remote_server_cannot_read_local_path`。

## 1. 上云目标

云端提供：

- 一个 MFS server endpoint。
- Source 管理。
- 后台同步。
- MFS Index 更新和缓存。
- 远程对象读取。
- 远程 `search/grep`。
- Job 状态和失败排查。

用户看到的是：

```bash
mfs profile add prod --url https://mfs.example.com --kind remote
mfs profile use prod
mfs add slack://eng --config .mfs/sources/slack-eng.toml
mfs add github://product --repo org/product --config .mfs/sources/github-product.toml
mfs search "why did pricing limit change" slack://eng
```

如果用户在 remote profile 下运行：

```bash
mfs add ./repo
```

返回明确错误，提示使用 local profile 处理本机文件，或改用 source URI。

## 2. 业务对象

```text
┌───────────────────────────────────────────────────────────────┐
│                          MFS Server                           │
│                                                               │
│  ┌────────────┐   ┌────────────────┐   ┌──────────────────┐   │
│  │ Source     │   │ IndexArtifact  │   │ Job              │   │
│  │ root/config│   │ kind/freshness │   │ sync/export/read │   │
│  └─────┬──────┘   └───────┬────────┘   └────────┬─────────┘   │
│        │                  │                     │             │
│        v                  v                     v             │
│  ┌────────────┐   ┌────────────────┐   ┌──────────────────┐   │
│  │ Object     │   │ SearchRecord   │   │ Audit/Event      │   │
│  │ uri/media  │   │ uri+locator    │   │ operation log    │   │
│  └────────────┘   └────────────────┘   └──────────────────┘   │
└───────────────────────────────────────────────────────────────┘
```

基础云端模型使用单租户或单组织。多 workspace、成员角色、组织计费属于商业化接口族。

## 3. 部署架构

```text
┌──────────────────────────── ingress ───────────────────────────────┐
│                         HTTPS / JSON API                           │
└───────────────────────────────┬────────────────────────────────────┘
                                │
                                v
┌────────────────────────────────────────────────────────────────────┐
│                            MFS API                                 │
│                                                                    │
│  /v1/add  /v1/sources  /v1/objects  /v1/search  /v1/grep  /v1/jobs │
└───────────────┬──────────────────────────────┬─────────────────────┘
                │                              │
                v                              v
┌────────────────────────────┐      ┌────────────────────────────────┐
│       Postgres             │      │          Worker Pool            │
│ sources / index / jobs     │      │ sync / artifacts / retrieval    │
│ objects / audit events     │      └───────────────┬────────────────┘
└──────────────┬─────────────┘                      │
               │                                    │
               v                                    v
┌────────────────────────────┐      ┌────────────────────────────────┐
│ Object Storage             │      │ Search Backend                  │
│ cache / extracted content  │      │ Milvus / Zilliz                 │
│ exports / pages            │      │ vector + BM25 metadata          │
└────────────────────────────┘      └────────────────────────────────┘
```

## 4. 部署方式

### 4.1 单容器 server

用于 demo、小规模自部署和快速试用。它把 API 和 worker 放在一个容器里，但仍然使用同一套 server 代码边界。

```bash
docker run --rm -p 8765:8765 ghcr.io/zilliztech/mfs-server:0.4.0
```

### 4.2 Docker Compose

用于本地开发、正式自部署和需要拆开 API/worker 的环境。

```bash
docker compose up -d postgres minio redis milvus mfs-api mfs-worker
```

### 4.3 单机生产

适合早期试用：

- 一个 VM。
- Postgres managed service。
- S3/R2/MinIO。
- Milvus 或 Zilliz Cloud。
- systemd 或 Docker Compose 管理 API/worker。

### 4.4 Kubernetes

用于更大规模部署：

```bash
helm install mfs ./charts/mfs \
  --set api.replicas=2 \
  --set worker.replicas=4 \
  --set objectStore.type=s3 \
  --set search.type=zilliz
```

## 5. 运维能力

必须有：

- source healthcheck。
- sync lag。
- MFS Index freshness。
- job retry。
- worker heartbeat。
- API latency。
- search latency。
- embedding cost。
- storage usage。
- operation log。

CLI 可见：

```bash
mfs profile status
mfs status
mfs status slack://eng
mfs status --diagnose slack://eng
mfs job list --failed
```

健康检查不再有独立命令，合并进 `mfs status <source-uri>`。

## 6. 镜像与包边界

| 交付物 | 定位 | 说明 |
| --- | --- | --- |
| PyPI `mfs-cli` | client | CLI、Python SDK、HTTP client、profile、输出 |
| PyPI `mfs-server` | server | API、local daemon、worker、engine、sources、objects、pipeline、storage |
| Docker `mfs-server` | 单容器 server | API + worker，适合 demo 和小规模 |
| Docker `mfs-api` | API server | 请求入口、job 创建、轻量 object read |
| Docker `mfs-worker` | worker | source sync、index artifacts、retrieval index、export、summary |

`mfs-api` 和 `mfs-worker` 共享 `mfs-server` package 是正常的。镜像边界按 entrypoint 和职责切，不按“完全不重复代码”切。

local daemon 是 `mfs-server` 的本机 entrypoint。Docker 用户如果想本地跑 server，使用 `mfs-server` image。

## 7. 人类准备项

业务上云阶段需要人类准备：

- 域名和 TLS。
- API token 或临时服务端 token。
- 数据库和对象存储。
- 队列服务。
- 向量后端。
- Secret manager。
- 日志和监控平台。

智能体可独立完成：

- Docker Compose。
- Helm chart 草案。
- API server skeleton。
- worker runner、queue adapter 和基础 job handler。
- 本地 fake source e2e。
- 文档和 runbook。

## 8. 商业化能力

这些能力使用独立产品线和接口族：

- 团队成员和角色。
- 多 workspace。
- 登录和认证体系。
- 私有 worker。
- 细粒度组织审计。
- 计费套餐。
- Web 控制台。
