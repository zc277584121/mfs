# 任务与协作型 Source 场景

任务/协作型 source 的对象通常是 issue、PR、ticket、database row、comment、review、status timeline。MFS 的目标是让 agent 能搜索这些对象，并打开完整上下文。

代表：

- Jira。
- Linear。
- GitHub Issues/PRs。
- Notion database。
- Trello。

## 1. GitHub Issues / PRs

### 添加 source

```bash
mfs add github://mfs --repo zilliztech/mfs --include issues,pulls --config .mfs/sources/github-mfs.toml
```

输出：

```text
Added source: github://mfs
type: github
health: ok
objects:
  github://mfs/issues.jsonl
  github://mfs/pulls.jsonl
  github://mfs/pulls.jsonl/{number}/diff
  github://mfs/pulls.jsonl/{number}/reviews
Sync started.
job: job_01HX...
```

### 浏览

```bash
mfs head -n 3 github://mfs/issues.jsonl
```

输出：

```jsonl
{"number":531,"title":"grep should respect path scope","state":"open","labels":["bug"],"updated_at":"2026-05-10T18:00:00Z"}
{"number":529,"title":"Add JSON output for tree","state":"closed","labels":["enhancement"],"updated_at":"2026-05-09T11:20:00Z"}
```

### 搜索

```bash
mfs search "path scoped grep bug" github://mfs/issues.jsonl
```

输出：

```text
[1] github://mfs/issues.jsonl  score=0.881
     issue: number=531
     title: grep should respect path scope
     state: open
     labels: bug
```

### 打开对象

```bash
mfs grep '"number":531' github://mfs/issues.jsonl
mfs export github://mfs/issues.jsonl ./issues.jsonl
jq 'select(.number == 531)' ./issues.jsonl
```

打开后包含结构化 issue 对象；comments、events、reviews 可以继续从对应对象过滤。

### PR diff 和 review

```bash
mfs grep '"number":42' github://mfs/pulls.jsonl
mfs cat github://mfs/pulls/42/diff.patch
mfs cat github://mfs/pulls/42/reviews.jsonl --range 0:100
```

行为：

- PR 主体、comments、reviews、checks、diff 分开暴露。
- 搜索可以把 PR 标题、描述、评论和 review 聚合为一个结果。
- 精确检查 diff 时用 `cat` 打开 diff 对象。

## 2. Jira

```bash
mfs add jira://platform --config .mfs/sources/jira-platform.toml
mfs tree jira://platform/projects -L 2
mfs search "billing rate limit rollout" jira://platform
mfs grep '"key":"MFS-123"' jira://platform/projects/MFS/issues.jsonl
```

输出：

```text
Issue MFS-123: Billing rate limit rollout
Status: In Progress
Assignee: chen
Priority: High
```

URI：

```text
jira://platform/projects/MFS/issues.jsonl
jira://platform/projects/MFS/sprints.jsonl
jira://platform/projects/MFS/versions.jsonl
```

## 3. Linear

```bash
mfs add linear://product --config .mfs/sources/linear.toml
mfs search "enterprise onboarding blocker" linear://product
mfs grep '"state":"Blocked"' linear://product/issues.jsonl
```

输出：

```jsonl
{"id":"LIN-44","title":"Enterprise onboarding blocker","state":"Blocked","assignee":"alice","team":"Growth"}
```

行为：

- Linear 的 team、project、cycle 都可以作为目录层。
- `issues` 对象适合脚本过滤。
- 搜索结果打开后展示 description、comments、links 和 state changes。

## 4. Notion database

```bash
mfs add notion://team --config .mfs/sources/notion-team.toml
mfs ls notion://team/databases
mfs cat notion://team/databases/roadmap/schema.json
mfs head -n 20 notion://team/databases/roadmap/records.jsonl
mfs search "Q3 roadmap vector search" notion://team/databases/roadmap
```

输出：

```text
[1] notion://team/databases/roadmap/records.jsonl  score=0.797
     page: id=pg_123
     title: Vector search improvements
     status: Planned
     owner: infra
```

行为：

- Notion database 更像结构化表，默认暴露 `schema` 和 `records`。
- 页面正文可以作为对象内容的一部分。
- relation、people、select 字段作为 metadata。

## 5. 任务/协作型 source 的默认行为

| 操作 | 行为 |
| --- | --- |
| `ls/tree` | 展示 project、team、database、对象集合 |
| `head/tail` | 查看 issues、pulls、comments、timeline |
| `cat --range A:B` | 区间读取评论、review、timeline |
| `grep` | 搜索标题、正文、评论和状态文本（可下推 provider search） |
| `search` | 返回对象级结果 |
| `export` | 把对象写本地后用 jq / Python / Pandas 处理 |

## 6. 用户最关心的问题

### 找为什么做了某个决定

```bash
mfs search "why did we change pricing limit" github://product
```

输出可能同时命中：

```text
[1] linear://product/issues.jsonl  score=0.842
    issue: id=LIN-88
[2] github://product/pulls.jsonl  score=0.809
    pull: number=312
[3] slack://eng/channels/pricing/2026-04-22/threads.jsonl  score=0.781
    thread: 1713780000.111
```

### 查某个 PR 的上下文

```bash
mfs grep '"number":312' github://product/pulls.jsonl
mfs cat github://product/pulls/312/diff.patch
mfs cat github://product/pulls/312/reviews.jsonl --range 0:100
```

### 查某个版本还有哪些 blocker

```bash
mfs export jira://platform/projects/MFS/issues.jsonl ./issues.jsonl
jq 'select(.fix_version == "v1.2" and .state != "Done")' ./issues.jsonl
```
