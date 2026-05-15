# 概念、URI、定位与命令决策

本文回答几个容易混淆的问题：本地路径和外部 source URI 怎么区分，`file:///...` 是什么，搜索结果如何定位具体对象，Source/Path/Object 怎么讲，MFS Index、分页和密度视图应该怎么设计。

## 1. 用户输入路径的规则

MFS 的日常命令接受两类输入。

本地文件用普通路径：

```bash
mfs cat ./docs/auth.md
mfs cat /data/repo/README.md
mfs search "retry policy" ./src
```

外部 source 用 URI：

```bash
mfs cat postgres://prod/public/tickets/schema.json
mfs head -n 20 postgres://prod/public/tickets/sample.jsonl
mfs tail -n 50 slack://eng/channels/incidents/2026-05-10/messages.jsonl
```

好处：

- 本地体验沿用普通文件路径。
- 外部 source 不和本地绝对路径冲突。
- Agent 一眼能看出数据来自哪个 source。
- 命令仍然是 file-like：`cat/head/tail/grep/search`，只是地址用 URI。

local profile 下，普通本地路径表示 daemon 所在机器的文件。通常 daemon 和 CLI 同机，所以体验就是「本地路径」。

remote profile 下，本地路径请求返回 `remote_server_cannot_read_local_path`。云端索引本地文件使用显式 upload API；server mounted path 使用显式 source 配置。

## 2. `file:///...` 的含义

`file:///data/repo/README.md` 是本地文件的标准 file URI。

拆解：

```text
scheme:    file
authority: empty
path:      /data/repo/README.md
```

MFS 用户文档里，本地路径直接写普通路径：

```bash
mfs cat ./README.md
mfs cat /data/repo/README.md
```

`file:///...` 只出现在：

- `--json` 输出里的规范化 `source` 字段。
- 跨进程、HTTP API、审计日志里需要规范化路径的场景。

相对路径保持用户输入的 `./docs/a.md` 或 `../repo/a.md`。

## 3. Source URI 与连接串的区别

MFS 的外部 source URI 是**用户操作地址**：

```text
postgres://prod/public/tickets/rows.jsonl
slack://eng/channels/incidents/2026-05-10/messages.jsonl
zendesk://support/tickets/records.jsonl
```

`postgres://prod`、`slack://eng`、`zendesk://support` 是 `mfs add` 注册出来的 source root。`prod`/`eng`/`support` 是 root URI 里的 alias；展示名放在 `label`。

数据库连接串是**凭据配置**：

```text
postgresql://user:password@host:5432/db
```

连接串只出现在 source TOML 或 secret provider 里。CLI、搜索结果、Skill 和 README 展示 source URI，不展示连接串。

## 4. 搜索结果的定位

搜索结果需要定位到具体 row、thread、issue 或 ticket。**定位信息放在结构化字段里；路径字符串保持可读、可复制、可继续操作**。

原因：

- DB row、Slack thread、Zendesk ticket 的定位字段不统一。
- `#...` 对 shell 用户不直观。
- JSON/JSONL 本来就适合脚本处理。
- file-like 的核心是「像文件一样读和过滤」，路径负责定位容器，locator 负责定位容器内对象。

搜索结果：

```json
{
  "source": "postgres://prod/public/tickets/rows.jsonl",
  "locator": {
    "kind": "row",
    "primary_key": {"id": 12}
  },
  "content": "Login broken after SSO migration"
}
```

人类输出：

```text
[1] postgres://prod/public/tickets/rows.jsonl  score=0.842
    row: id=12
    subject: Login broken after SSO migration
```

对单个业务对象天然有稳定路径的 source，可以直接暴露对象路径：

```text
linear://eng/teams/ENG/issues.jsonl/ENG-123
github://mfs/pulls/42/reviews.jsonl
zendesk://support/tickets/records.jsonl
```

数据库表不默认生成 `rows/id=12` 这种伪目录，因为 row 不是文件、表也不是真目录。`rows` 是一个 JSONL 形态的对象，row 用 `locator` 表达。

## 5. Source、Path、Object

用户文档只讲三个概念。

```text
Source  = 一个已配置的数据源
Path    = MFS 可以读取、搜索或列出的地址（本地路径或 source URI）
Object  = Path 指向的对象：文件、JSONL、record、thread、ticket、row 或索引表示
```

例：

```text
Source:  postgres://prod
Path:    postgres://prod/public/tickets/rows.jsonl
Object:  rows 这个 JSONL 对象；其中 id=12 的 row 由 locator 指向
```

**不引入"视图"概念**。一个 source 在 URI 树下暴露一组对象——`schema`、`sample`、`rows`、`messages`、`records` 等。用户视角里它们就是「source 下面的对象」，cat 它就能读。每类 source 暴露哪些对象写在该 source 的文档里，作为稳定契约；agent 通过 `mfs ls` / `mfs source inspect` 发现。

参考：Mirage 把每个 source mount 成一个目录、目录下的内容由 source 决定，并由对象自己决定 `cat` 时怎么渲染——这与我们的方向一致。我们不用绝对路径 mount（避免和本地路径冲突），改用 `scheme://` 表达 source 身份；其余思路相同。

## 6. MFS Index 的定位

详细定义见 [17-source-sync-and-mfs-index.md](17-source-sync-and-mfs-index.md)。本节给出用户面口径。

**MFS Index = 一个 source 在 MFS 内的全部索引集合**。分两层：

- **Index Artifacts**：metadata、normalized content、structure、chunks、summary、JSONL page cache 等可 `cat / head / tail` 的产物。
- **Retrieval Index**：向量 + BM25，给 `search / grep` 用。

用户面只用 **MFS Index** 一个名字：

```bash
mfs add ./repo
mfs add ./repo --force
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
mfs add postgres://prod
mfs add postgres://prod --force
mfs status postgres://prod
```

`mfs status` 显示 `index=fresh/stale`，指 Artifacts 的 freshness 汇总。

开发者面才出现 Artifacts / Retrieval Index / Index Plan 三个细分。

排查 Retrieval Index 细节用：

```bash
mfs status --verbose postgres://prod
mfs status --diagnose postgres://prod
```

## 7. 分页设计

大对象读取**不引入 cursor token**。理由：

- cursor 是 stateful 复杂度，agent 难管，token 过期/兼容性问题多。
- 真要遍历大对象用 `mfs export` 物化一份。
- 大对象的过滤用 `mfs grep`（server-side pushdown）。
- 增量数据用 `tail -f`。
- DB query 结果不稳定的问题靠 `export` 物化解决，不该在 cat 里隐式承担。

统一接口：

```bash
mfs head -n N <uri>          # 固定前 N 行/记录，无状态
mfs tail -n N <uri>          # 固定后 N 行/记录，无状态
mfs tail -f <uri>            # 流式跟随，仅 append-only source
mfs cat <uri>                # 完整对象；大对象拒绝并提示
mfs cat <uri> --range A:B    # 按行/记录区间读取
mfs export <uri> <file>      # 完整导出到本地
```

`head/tail/cat` 职责完全不重叠：

- `head/tail` 只看端点，不带范围。
- `cat` 默认完整；`--range A:B` 取区间；都不带 cursor。

大对象 `cat` 错误模板：

```text
Object is too large for full cat: postgres://prod/public/tickets/rows.jsonl
size_hint: 4.2GiB
try:
  mfs head -n 20 postgres://prod/public/tickets/rows.jsonl
  mfs cat postgres://prod/public/tickets/rows.jsonl --range 0:1000
  mfs export postgres://prod/public/tickets/rows.jsonl ./tickets.jsonl
```

`--range` 的含义按对象类型决定：

| 对象类型 | `--range A:B` 的单位 |
| --- | --- |
| 文本文件 | 行号 |
| JSONL | record 索引 |
| CSV | row 索引（不含 header） |
| 二进制 | 不支持，报错 |

range 是闭开区间 `[A, B)`。`A` 省略表示 0；`B` 省略表示末尾。

## 8. 密度视图与 head/tail 的分工

`head/tail` 是精确语义：

```bash
mfs head -n 20 <uri>
mfs tail -n 50 <uri>
```

`--peek/--skim/--deep` 是密度视图，给 agent 快速理解结构和上下文：

```bash
mfs tree --peek ./repo
mfs ls --skim ./docs
mfs cat --skim ./README.md
mfs cat --peek postgres://prod/public/tickets/schema.json
```

| 命令 | 用途 |
| --- | --- |
| `head / tail` | 精确、可脚本化、行/record 语义 |
| `peek / skim / deep` | 给 agent 快速理解结构（heading、symbol、目录） |

**密度视图主要面向 document/code 对象**，对结构化 source 的退化行为：

| 对象类型 | `--peek` | `--skim` | `--deep` |
| --- | --- | --- | --- |
| document（md / pdf / docx） | heading skeleton | + 每段首句 | 展开全文 |
| code | symbol tree | + signature/docstring | 展开关键 region |
| schema 对象 | 顶层结构 | + 字段类型 | 展开全部字段 |
| sample / rows / messages / records | 等价 `head -n N`，N 由密度档决定 | 同上 | 不支持，建议改用 `cat --range` |
| 二进制 | 仅 metadata | + media type 描述 | 不支持 |

`tree --peek/--skim/--deep` 对无界 source（Slack 全 workspace、S3 bucket 根）有最大列举数限制；超过时输出截断提示，引导用户加 path 缩小范围。

## 9. Source root、alias、label

`mfs add` 注册外部 source 时，主参数是 source root URI：

```bash
mfs add postgres://prod --config .mfs/sources/prod-postgres.toml
```

日常命令：

```bash
mfs ls postgres://prod
mfs search "login failure" postgres://prod/public/tickets
mfs add postgres://prod --force
```

概念分工：

```text
source_id    内部稳定 ID，例如 src_01HX...
source root  用户操作根地址，例如 postgres://prod
scheme       source 类型，例如 postgres
alias        root URI 里的 prod，在 workspace 内唯一
label        可选展示名，例如 Production Postgres
```

`alias` 进入脚本、Skill、搜索结果；`label` 只用于展示，不参与路径解析。

## 10. Profile 的角色

`profile` 决定 client 连哪个 endpoint。**`kind` 字段决定行为，名字只是 alias**：

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

**`kind = "local"` 的定义**：CLI 进程和 daemon **共享同一文件系统命名空间**。同一字符串路径在两边都能解析到同一文件。

这意味着以下场景**不算 local，应该按 remote 处理**：

- daemon 跑在 Docker 容器，CLI 在 host（除非显式 bind mount 整个相关目录）。
- daemon 跑在 WSL，CLI 在 Windows host。
- daemon 跑在 SSH 远端 box，CLI 在笔记本（即使用 port forward 把 127.0.0.1 指过去）。

profile 管理命令：

```bash
mfs profile add local  --url http://127.0.0.1:8765 --kind local
mfs profile add prod   --url https://mfs.example.com --kind remote
mfs profile use prod
mfs profile list
mfs profile status
```

`mfs daemon` 和 `mfs profile` 是正交概念：daemon 决定本机是否跑 server 进程；profile 决定 client 连哪个 endpoint。
