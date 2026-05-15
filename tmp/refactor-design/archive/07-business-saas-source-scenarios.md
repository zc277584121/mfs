# 业务 SaaS Source 场景

业务 SaaS source 的难点是对象模型复杂、权限强、客户自定义字段多。MFS 的用户体验应该先暴露对象和记录，再用 preset 和 source 配置帮助搜索。

代表：

- Salesforce。
- HubSpot。
- Zendesk。

## 1. Zendesk

### 添加 source

```bash
mfs add zendesk://support --config .mfs/sources/zendesk-support.toml
```

输出：

```text
Added source: zendesk://support
type: zendesk
health: ok
objects:
  zendesk://support/tickets/schema.json
  zendesk://support/tickets/records.jsonl
  zendesk://support/tickets/comments.jsonl
  zendesk://support/users/records.jsonl
Sync started.
job: job_01HX...
```

### 看 tickets

```bash
mfs head -n 3 zendesk://support/tickets/records.jsonl
```

输出：

```jsonl
{"id":123,"subject":"Login broken after SSO migration","status":"open","priority":"high","updated_at":"2026-05-10T12:30:00Z"}
{"id":124,"subject":"Refund request for annual plan","status":"pending","priority":"normal","updated_at":"2026-05-10T13:00:00Z"}
```

### 搜索

```bash
mfs search "enterprise customer cannot login with SSO" zendesk://support/tickets
```

输出：

```text
[1] zendesk://support/tickets/records.jsonl  score=0.856
     ticket: id=123
     subject: Login broken after SSO migration
     status: open
     priority: high
```

### 打开 ticket

```bash
mfs grep '"id":123' zendesk://support/tickets/records.jsonl
mfs grep '"ticket_id":123' zendesk://support/tickets/comments.jsonl
mfs export zendesk://support/tickets/records.jsonl ./tickets.jsonl
jq 'select(.id == 123)' ./tickets.jsonl
```

打开后包含结构化对象，适合 agent 继续过滤、转成 DataFrame 或导出。

行为：

- `records` 包含 provider 记录结构。
- 单对象查看优先用 `grep` 过滤；定位信息也放在搜索结果的 `locator`。
- 附件作为可打开 URI。

## 2. Salesforce

URI：

```text
salesforce://sales/accounts/schema.json
salesforce://sales/accounts/records.jsonl
salesforce://sales/accounts/activities.jsonl
salesforce://sales/opportunities/schema.json
salesforce://sales/opportunities/records.jsonl
```

命令：

```bash
mfs add salesforce://sales --config .mfs/sources/salesforce.toml
mfs cat salesforce://sales/accounts/schema.json
mfs head -n 20 salesforce://sales/accounts/records.jsonl
mfs search "renewal risk for Acme" salesforce://sales
```

输出：

```text
[1] salesforce://sales/accounts/records.jsonl  score=0.821
     account: id=001xx000003DGbY
     name: Acme Corp
     segment: Enterprise
     renewal_date: 2026-06-30
     health_score: red
```

打开：

```bash
mfs grep '"Id":"001xx000003DGbY"' salesforce://sales/accounts/records.jsonl
```

行为：

- Salesforce 自定义字段很多，搜索字段必须可在 source TOML 配置。
- preset 可以给 Account、Opportunity、Case 的默认字段。
- 权限跟随 Salesforce 用户或 integration user。

## 3. HubSpot

```bash
mfs add hubspot://crm --config .mfs/sources/hubspot.toml
mfs ls hubspot://crm
mfs head -n 20 hubspot://crm/companies/records.jsonl
mfs search "companies interested in enterprise plan" hubspot://crm
```

输出：

```text
[1] hubspot://crm/companies/records.jsonl  score=0.790
     company: id=901
     name: Example Inc
     lifecycle_stage: salesqualifiedlead
     owner: bob
```

## 4. 业务 SaaS 的 Index Plan 配置

Zendesk tickets：

```toml
[index.tickets]
object = "tickets"
primary_key_fields = ["id"]
updated_at_field = "updated_at"
include_comments = true

[index.tickets.retrieval]
text_fields = ["subject", "description", "latest_public_comment", "latest_internal_note"]
metadata_fields = ["id", "status", "priority", "requester_id", "assignee_id", "updated_at"]
```

Salesforce accounts：

```toml
[index.accounts]
object = "Account"
primary_key_fields = ["Id"]
updated_at_field = "SystemModstamp"

[index.accounts.retrieval]
text_fields = ["Name", "Description", "Customer_Notes__c", "Renewal_Risk__c"]
metadata_fields = ["Id", "OwnerId", "Industry", "Renewal_Date__c", "Health_Score__c"]
```

## 5. 业务 SaaS 的 UX 原则

- 首先暴露 provider 的对象和记录，不隐藏复杂性。
- `schema` 要显示标准字段和自定义字段。
- 搜索字段来自 preset 加用户配置。
- 搜索结果返回 URI 和 locator；用户用 `grep`、`cat --range` 或 `export + jq` 打开对象。
- `--json` 输出包含原始字段。
- 搜索结果遵守 provider 权限和字段可见性。
- 同步要处理删除、归档、状态变化和评论/活动变化。
