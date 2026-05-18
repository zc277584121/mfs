# Agent Skill 指南

本文写给**给 LLM agent 集成 MFS 的人**（Skill 作者、agent framework 维护者、prompt 工程师）。教 agent 何时用哪个命令、怎么解读返回结果、如何避开常见坑。

不是给 connector 贡献者看的（那个看 [07-contributing-connector.md](07-contributing-connector.md)），也不是给最终用户看的（看 README）。

## 1. Agent 的 MFS 心智模型

让 agent 先建立这套心智，再讨论命令：

```
Agent 想找信息
   │
   ├─ 已经知道路径   →  cat / head / tail / ls / tree
   │
   ├─ 知道关键词     →  grep
   │
   ├─ 概念性问题     →  search（语义混合）
   │
   └─ 不知道有什么   →  tree --peek 先扫一圈
```

**核心规则**：

1. **看到 URI 不要瞎猜**——先 `mfs ls <uri>` 或 `mfs connector inspect <root>` 了解 connector 暴露什么对象。
2. **结果是可继续操作的**——search/grep 返回的 `source` URI 直接喂给 cat / head / export。
3. **大对象不要 cat**——`mfs cat <uri>` 对大对象会拒绝，要用 head/tail/range/export。
4. **结构化对象不要用 `--peek/--skim/--deep`**——这些只对 document/code 形态有效，对 JSONL 报错。

## 2. 推荐工作流

### 工作流 A：在一个 connector 内找东西

```bash
# 1. 知道大致范围，先了解 connector 暴露什么
mfs connector inspect postgres://prod
# 或：
mfs tree postgres://prod --peek -L 2

# 2. 看具体目录下有什么对象
mfs ls postgres://prod/public/tickets
# schema.json / rows.jsonl 两个对象

# 3. 看 schema 理解数据
mfs cat postgres://prod/public/tickets/schema.json

# 4. 看几条样本理解数据形状
mfs head -n 5 postgres://prod/public/tickets/rows.jsonl

# 5. 语义搜索找候选
mfs search "customer cannot login" postgres://prod/public/tickets --top-k 5

# 6. 精确读单条
mfs grep '"id":12' postgres://prod/public/tickets/rows.jsonl
```

### 工作流 B：在本地代码仓库里找 bug

```bash
mfs tree --peek -L 2 ./src              # 了解结构
mfs search "session expiration" ./src   # 语义找候选
mfs grep "ERR_TOKEN" ./src              # 关键词找精确位置
mfs cat ./src/auth/token.py -n 150:180  # 读上下文
```

### 工作流 C：跨多个 connector 找一个决策的来龙去脉

```bash
mfs search "why did we change pricing limit" --all --top-k 10
# 返回可能混合：linear issue / github PR / slack thread

# 拿到结果继续展开
mfs cat linear://product/teams/Pricing/issues.jsonl
mfs grep '"id":"LIN-88"' linear://product/teams/Pricing/issues.jsonl
```

### 工作流 D：跟随实时数据流

```bash
mfs tail -f slack://eng/channels/incidents/today/messages.jsonl
mfs tail -f s3://logs/app/today.jsonl
```

## 3. 怎么解读返回结果

### `--json` envelope

每个命令都有 `--json` 输出。agent 优先用 `--json` 而不是解析人类输出。统一结构：

```json
{
  "source": "postgres://prod/public/tickets/rows.jsonl",
  "lines": null,
  "content": "Login broken after SSO migration",
  "score": 0.842,
  "locator": {
    "schema": "public",
    "table": "tickets",
    "pk": {"id": 12}
  },
  "metadata": {
    "kind": "search",
    "chunk_kind": "row_text",
    "connector_type": "postgres",
    "media_type": "application/x-ndjson",
    "fields": {
      "status": "open",
      "priority": "high"
    }
  }
}
```

agent 关心的字段：

| 字段 | 怎么用 |
|---|---|
| `source` | 喂给下一个命令（cat / head / grep / export） |
| `content` | 给 LLM 看的内容 snippet |
| `locator` | 当 source 是集合对象时，精确指向单条 record |
| `score` | 排序参考；低于 0.5 通常不可靠 |
| `metadata.chunk_kind` | 区分召回类型（body / row_text / thread_aggregate / vlm_description / summary） |
| `metadata.media_type` | 判断对象类型，决定下一步用什么命令 |
| `metadata.fields` | 不打开对象就能看到的业务字段（status / priority 等） |

### 从结果回到对象的两种方式

如果 `lines` 不为 null（document / code）：

```bash
mfs cat <source> -n <start>:<end>      # 直接读那段
```

如果 `locator` 不为 null（DB row / issue / ticket / thread）：

```bash
# 方式 A：用 grep 找单条
mfs grep '"id":12' <source>

# 方式 B：导出后过滤（数据量大或要复杂查询）
mfs export <source> /tmp/data.jsonl
jq 'select(.id == 12)' /tmp/data.jsonl
```

## 4. 反模式：不要这样做

| ❌ 反模式 | ✅ 推荐 |
|---|---|
| `mfs cat <huge-rows.jsonl>`（不带 `--range`） | `mfs head -n 20` 或 `mfs cat --range 0:100` |
| `mfs cat --peek <rows.jsonl>` | `mfs head -n 5 <rows.jsonl>` |
| 拼构造 path 取单 record（如 `tickets/12.json`） | 用 search/grep 的结果 + `locator` |
| 用 pipe 传递 source 元信息 | 直接传 path 参数：`mfs search "..." <path>` |
| `mfs add <uri>` 然后假设立刻 search 可用 | `mfs status <uri>` 看 sync 进度，等 search=available |
| 用 `mfs cat` 看图片 | `mfs cat <img> --meta` 看 VLM description |
| 在 `--all` 上跑没有 filter 的 query | 加 `--top-k` 限制 / 加 path 缩小范围 |

## 5. 错误码处理

agent 拿到 `--json` 输出里的 error 时，按 `code` 字段决定怎么 recover：

| code | 意思 | agent 该怎么办 |
|---|---|---|
| `object_too_large_for_cat` | 文件太大，cat 拒绝 | 改用 `head` / `tail` / `cat --range` / `export`，按 suggestions |
| `is_directory` | 对目录 cat | 改用 `ls` / `tree` |
| `sync_already_running` | 同 connector 正在 sync | `mfs status <uri>` 看进度，或 `mfs job cancel` |
| `connector_removing` | connector 正在 remove | 等清理完，或换 connector |
| `connector_unhealthy` | connector 连不上 | 看 error.details；用户层凭据问题 |
| `tail_unsupported` | connector 不支持 tail -f | 用 `head -n N` 拿快照 |
| `density_unsupported` | 结构化对象不能用 `--peek/--skim/--deep` | 改用 `head` |
| `since_unsupported` | connector 不支持 `--since` | 去掉 `--since`，直接 `mfs add` |
| `range_unsupported` | 二进制对象不支持 `--range` | 用 `head -c` 字节或 `export` |
| `chunk_max_exceeded` | 对象太大，部分索引 | search 仍然可用但召回不全；建议用户加 `filter_expr` |
| `remote_server_cannot_read_local_path` | remote profile 收到本地 path | 切 local profile，或用 source URI |
| `field_missing` | text_fields 配的字段不存在 | 用户层配置问题；提示用户改 connector TOML |

所有错误都有 `suggestions` 字段——优先按 suggestion 行动，不要试错。

## 6. Skill 文件应该有什么

写一个 `skill.md` / SKILL 文档给 agent 时，建议包含：

1. **MFS 是什么**（1 段）：file-like shell-native CLI，agent 直接用 shell 命令搜/读各种数据源
2. **命令清单 + 用途**（一张表）：参考 [03-cli-commands.md](03-cli-commands.md)
3. **推荐工作流**（本文 §2 那几个）
4. **结果 envelope** + 怎么从结果回到对象（本文 §3）
5. **反模式列表**（本文 §4）
6. **错误码处理**（本文 §5）
7. **本 agent 配置的 connector 清单**（运行时 `mfs connector list --json` 注入）

skill 文件不需要包含贡献者文档、架构文档——那些跟 agent 推理无关。

## 7. 让 agent 自己发现能力

agent 不要硬编码"什么 connector 支持什么操作"。运行时 query：

```bash
mfs connector list --json
mfs connector inspect <root> --json     # 拿到该 connector 的 PROMPT / capabilities / 暴露的对象
mfs stat <uri> --json                   # 单个对象的 capabilities (cat / grep / tail / range)
```

`capabilities` 字段告诉 agent 这个对象能用什么命令：

```json
{
  "cat": "denied_unless_range",
  "grep": "pushdown",
  "tail": false,
  "range": true
}
```

agent 看到 `cat="denied_unless_range"` 就不要直接 cat，直接走 head 或 range。看到 `tail=false` 就不要试 `tail -f`。

这种动态发现让 agent **跟着新 connector 自动适应**——不用每加一个 source 就改 skill 文件。

## 8. 多步骤任务的中断与恢复

agent 跑长流程（如先 mfs add 一个大 connector 再 search）：

- `mfs add` 默认后台跑，立刻返回 `job_id`
- agent 应该 poll `mfs status <uri>` 或 `mfs job inspect <job_id>` 直到 `search: available`
- 不要假设 add 完立刻能 search

poll 模板（伪 shell）：

```bash
JOB=$(mfs add postgres://prod --config x.toml --yes --json | jq -r '.job_id')
while true; do
  STATUS=$(mfs job inspect "$JOB" --json | jq -r '.status')
  case $STATUS in
    succeeded) break ;;
    failed|cancelled) exit 1 ;;
    *) sleep 30 ;;
  esac
done
mfs search "..." postgres://prod
```

或者直接 `mfs add <uri> --sync`（前台等同步完成，到 v0.4 实现时确认是否支持）。

## 9. 给 connector 贡献者的提示

如果你贡献了一个新 connector，别忘了在 connector 目录里写一份 `PROMPT.md`，agent skill 会读它生成上下文。详见 [07-contributing-connector.md §5](07-contributing-connector.md#5-promptmd-范本)。

好的 `PROMPT.md` 让 agent 不需要试错就知道：

- 这个 connector 下面有什么对象
- 每个对象 cat 出来是什么格式
- 大对象有什么限制
- 推荐什么命令路径
