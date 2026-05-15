# 本地文件与代码仓库场景

本地文件是 MFS 的基础场景，也是最顺手的入口。

## 1. 工作目录索引

```bash
mfs daemon start
mfs profile use local
mfs add .
```

输出：

```text
Processing 184 files under /repo
Indexed: 184 files scanned, 184 touched, 0 deleted, 2341 chunks queued.
Worker running in background. Run `mfs status` to check progress.
```

行为：

- CLI 通过 HTTP 把 path 和选项发给 local daemon（`POST /v1/add`）。
- local daemon 读取本机目录并执行 file source。
- 读取 ignore 规则。
- 跳过二进制、缓存目录和不可索引文件。
- 生成 chunk、embedding 和关键词索引。
- 状态写入 MFS 本地状态目录。
- 项目目录不写入额外文件。

## 2. 同步等待完成

```bash
mfs add . --sync
```

输出：

```text
Processing 184 files under /repo
Embedding 2341 chunks...
Embedding complete.
```

适合 CI、测试和需要立即搜索的 agent 任务。

## 3. 强制重建

```bash
mfs add . --force
```

行为：

- 重新扫描目标路径。
- 重新计算 hash 和 chunk。
- 删除已经不存在的文件对应索引。
- 适合怀疑索引状态异常时使用。

## 4. Watch 工作流

```bash
mfs add . --watch --interval 60s
```

输出：

```text
Watching 1 path(s) (interval=60000ms). Ctrl+C to stop.
[12:01:31] Change detected, re-indexing...
Indexed: 3 files scanned, 1 touched, 0 deleted, 12 chunks queued.
```

行为：

- 适合持续写日志、记忆文件、代码编辑。
- watch 事件只作为触发信号，仍然做可靠扫描对齐。
- watch、manifest、mtime/hash 对比都在 local daemon 内执行。
- 查看正在 watch 的目录用 `mfs status --watch`。
- 停止 watch 直接 Ctrl+C，或者 `mfs remove <path>` 同时删除索引。

## 5. 搜索代码问题

```bash
mfs search "where is queue retry handled" . --top-k 5
```

输出：

```text
[1] src/mfs/ingest/worker.py  score=0.891
116  def run_worker_once(queue: TaskQueue, store: MilvusStore) -> int:
117      task = queue.pop_next()
118      if task is None:

[2] src/mfs/ingest/queue.py  score=0.804
 73  class QueueTask:
 74      attempts: int = 0
```

继续查看：

```bash
mfs cat --skim src/mfs/ingest/worker.py
mfs cat -n 100:150 src/mfs/ingest/worker.py
```

## 6. 精确搜索错误码

```bash
mfs grep -C 2 "ERR_TOKEN_EXPIRED" ./src
```

输出：

```text
src/auth/token.py
165  if token.expired:
166      logger.info("refresh token expired")
167      raise TokenExpiredError("ERR_TOKEN_EXPIRED")
168  return token
```

行为：

- 已索引文件优先走关键词索引。
- 未索引但本地存在的文件可以使用系统 grep fallback。

## 7. 浏览目录结构

```bash
mfs tree --peek -L 2 .
```

输出：

```text
.
├── docs
│   ├── getting-started.md
│   └── search-and-inspect.md
├── src
│   └── mfs
└── tests
```

```bash
mfs ls --skim ./docs
```

输出：

```text
docs/
  getting-started.md
    Install, configure embedding provider, index a directory, and run search.
  search-and-inspect.md
    Explains the search plus inspect workflow for agents.
```

## 8. 读取文件

```bash
mfs cat --peek ./docs/getting-started.md
mfs cat --skim ./docs/getting-started.md
mfs cat --deep ./docs/getting-started.md
mfs cat -n 40:90 ./docs/getting-started.md
```

行为：

- `--peek` 看结构。
- `--skim` 看结构和摘要。
- `--deep` 看更完整内容。
- `-n` 精确读行范围。

## 9. GitHub repo 场景

GitHub repo 有两种入口。

### 9.1 本地 clone

```bash
git clone https://github.com/zilliztech/mfs.git
cd mfs
mfs add .
mfs search "density preset" .
```

这是本地代码搜索的最短路径。

### 9.2 远程 GitHub source

```bash
mfs add github://mfs --repo zilliztech/mfs --branch main --config .mfs/sources/github-mfs.toml
mfs tree github://mfs -L 2
mfs cat github://mfs/README.md
mfs search "density preset" github://mfs
```

输出：

```text
[1] github://mfs/src/mfs/search/density.py  score=0.876
 12  class DensityParams:
 13      width: int
```

行为：

- 远程 GitHub source 暴露 repo 文件树。
- 默认读取 branch head 内容。
- Issues/PRs 归到任务/协作型 source，见 [05-task-and-collaboration-source-scenarios.md](05-task-and-collaboration-source-scenarios.md)。
- 再同步：`mfs add github://mfs`（幂等）。

## 10. 用户体验底线

- 本地目录是最快上手路径。
- README 首屏命令从本地 path 开始。
- 搜本地文件不需要先写 source 配置。
- 搜索结果必须能继续 `cat`、`grep`、`tree` 和 line range 检查。
