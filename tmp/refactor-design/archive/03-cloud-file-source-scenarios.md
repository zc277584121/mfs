# 云端文件型 Source 场景

文件型 source 天然接近 MFS 的本地文件体验。用户希望看到目录、文件、文档内容和附件。

代表：

- Google Drive。
- Feishu Docs。
- S3/R2/OSS/GCS/MinIO。
- Dropbox、Box。

## 1. Google Drive

### 添加 source

```bash
mfs add gdrive://company-docs --config .mfs/sources/gdrive.toml
```

输出：

```text
Added source: gdrive://company-docs
type: gdrive
health: ok
Sync started.
job: job_01HX...
```

### 浏览

```bash
mfs tree gdrive://company-docs -L 2
```

输出：

```text
gdrive://company-docs
├── Product
│   ├── Pricing Strategy.gdoc
│   └── Roadmap 2026.gsheet
├── Support
│   └── Enterprise FAQ.gdoc
└── Legal
    └── DPA.pdf
```

### 读取文档

```bash
mfs cat --skim 'gdrive://company-docs/Product/Pricing Strategy.gdoc'
```

输出：

```text
Pricing Strategy
  Current packaging separates self-serve and enterprise tiers.

Discount Rules
  Enterprise discounts require approval above 25%.
```

行为：

- Google Docs 可导出为文本视图。
- PDF 使用文档转换链路抽取文本。
- Google Sheets 默认提供表格 representation：

```bash
mfs ls 'gdrive://company-docs/Product/Roadmap 2026.gsheet'
mfs head -n 20 'gdrive://company-docs/Product/Roadmap 2026.gsheet/Sheet1'
```

## 2. Feishu Docs

```bash
mfs add feishu://team-docs --config .mfs/sources/feishu.toml
mfs search "客户续费风险" feishu://team-docs
mfs cat --skim feishu://team-docs/销售/续费流程.docx
```

输出：

```text
[1] feishu://team-docs/销售/续费流程.docx  score=0.812
  续费风险需要在到期前 45 天进入 review。
```

行为：

- 中文内容要正常分词、grep 和搜索。
- 权限摘要跟随 Feishu 文档可见性。
- 飞书表格和多维表格优先暴露 JSONL/CSV representation。

## 3. S3 / 对象存储

### 添加 source

```bash
mfs add s3://logs --config .mfs/sources/s3-logs.toml
```

配置示例：

```toml
[source]
type = "s3"
root = "s3://logs"
label = "Application Logs"
credential_ref = "secret:aws-logs-readonly"

[s3]
bucket = "company-logs"
prefixes = ["app/", "billing/"]
include = ["**/*.jsonl", "**/*.log", "**/*.csv"]
```

### 使用

```bash
mfs tree s3://logs/app/2026/05 -L 2
mfs grep "ERR_TIMEOUT" s3://logs/app/2026/05/10/app.jsonl
mfs head -n 5 s3://logs/app/2026/05/10/app.jsonl
mfs tail -f s3://logs/app/today
mfs export s3://logs/app/2026/05/10/app.jsonl ./app-2026-05-10.jsonl
```

S3 上真实文件保留原文件名和后缀（`app.jsonl`、`report.csv`）。MFS 为非文件 source 生成的虚拟对象也带 media type 后缀，跟真实文件视觉一致。

输出：

```text
s3://logs/app/2026/05/10/app.jsonl
8842  {"level":"error","code":"ERR_TIMEOUT","request_id":"r_123","message":"upstream timeout"}
```

行为：

- 大对象不默认整体读入。
- `head`、`tail`、`cat --range` 支持区间读取。
- `tail -f` 跟随 append-only 对象（声明 `efficient_tail = true` 的 source）。
- `grep` 可以使用对象存储 select、缓存扫描或流式扫描，取决于后端能力。

## 4. 文件型 source 的默认行为

| 操作 | 行为 |
| --- | --- |
| `ls/tree` | 展示目录和文件 |
| `cat` | 读取文本文件、转换文档或提示二进制 |
| `search` | 搜索已索引文本、文档、代码和可抽取内容 |
| `grep` | 精确匹配文本，必要时流式扫描 |
| `head/tail` | 查看 JSONL、CSV、长日志和表格头尾 |
| `cat --range A:B` | 区间读取大对象 |
| `tail -f` | 跟随 append-only 对象 |
| `export` | 把外部对象写成本地文件 |

## 5. 搜索结果打开方式

```text
[1] gdrive://company-docs/Product/Pricing Strategy.gdoc  score=0.818
```

继续查看：

```bash
mfs cat --skim 'gdrive://company-docs/Product/Pricing Strategy.gdoc'
mfs cat -n 20:60 'gdrive://company-docs/Product/Pricing Strategy.gdoc'
```

如果对象是 JSONL：

```text
[1] s3://logs/app/2026/05/10/app.jsonl  score=0.701
    record: 8842
```

继续定位：

```bash
mfs cat s3://logs/app/2026/05/10/app.jsonl --range 8800:8900
mfs grep "r_123" s3://logs/app/2026/05/10/app.jsonl
```
