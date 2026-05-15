# 消息型 Source 场景

消息型 source 的核心问题是：单条消息太碎，直接当成文件不好用。用户体验应该围绕 channel、thread、day、session 和附件。

代表：

- Slack。
- Discord。
- Gmail。
- 飞书群聊。
- Telegram。
- Email。

## 1. Slack

### 添加 source

```bash
mfs add slack://eng --config .mfs/sources/slack-eng.toml
```

输出：

```text
Added source: slack://eng
type: slack
health: ok
channels: 12
Sync started.
job: job_01HX...
```

### URI 形态

每个 source 决定自己的对象布局；对象名带 media type 后缀：

```text
slack://eng/channels/general/2026-05-10/messages.jsonl
slack://eng/channels/general/2026-05-10/threads.jsonl
slack://eng/channels/incidents/2026-05-10/messages.jsonl
slack://eng/users
slack://eng/files/{file_id}
```

### 查看当天聊天

```bash
mfs cat slack://eng/channels/incidents/2026-05-10/messages.jsonl --range 0:5
```

输出：

```jsonl
{"ts":"1715320000.123","user":"U1","text":"deploy started","thread_ts":"1715320000.123"}
{"ts":"1715320060.456","user":"U2","text":"api timeout is rising","thread_ts":"1715320000.123"}
{"ts":"1715320120.789","user":"U3","text":"rollback is ready","thread_ts":"1715320000.123"}
```

跟随今天的频道：

```bash
mfs tail -f slack://eng/channels/incidents/today/messages.jsonl
```

### 搜索事件

```bash
mfs search "billing deploy caused timeout" slack://eng/channels/incidents
```

输出：

```text
[1] slack://eng/channels/incidents/2026-05-10/threads.jsonl  score=0.862
     thread: 1715320000.123
     12:00 U1 deploy started
     12:01 U2 api timeout is rising
     12:02 U3 rollback is ready
```

继续查看：

```bash
mfs grep "1715320000.123" slack://eng/channels/incidents/2026-05-10/messages.jsonl
mfs cat slack://eng/files/F123-error-chart.png
mfs export slack://eng/channels/incidents/2026-05-10/messages.jsonl ./incidents.jsonl
```

## 2. Discord

```bash
mfs add discord://community --config .mfs/sources/discord.toml
mfs tree discord://community/channels -L 2
mfs grep "installation failed" discord://community/channels/help
mfs search "users cannot install on windows" discord://community
```

输出：

```text
[1] discord://community/channels/help/2026-05-09/threads.jsonl  score=0.804
     thread: 982311
     user reported Windows installer permission error
```

行为：

- forum thread、channel day log、attachments 都是 URI 的一部分。
- 搜索默认按 thread 或时间窗口聚合。

## 3. Gmail / Email

URI：

```text
gmail://work/labels/inbox/2026-05/messages.jsonl
gmail://work/labels/support/2026-05/threads.jsonl
gmail://work/attachments/
```

命令：

```bash
mfs add gmail://work --config .mfs/sources/gmail-work.toml
mfs search "contract renewal from acme" gmail://work
mfs grep "abc123" gmail://work/labels/support/2026-05/threads.jsonl
```

输出：

```text
Subject: Re: Acme contract renewal
From: buyer@acme.com
Date: 2026-05-08

We need to confirm the enterprise renewal terms before Friday.
```

行为：

- 邮件搜索结果按 thread 返回，保持上下文完整。
- 附件作为可继续 `cat` 或 `export` 的 URI。
- metadata 包含 from、to、cc、subject、date、labels。

## 4. 飞书群聊

```bash
mfs add feishu-chat://company --config .mfs/sources/feishu-chat.toml
mfs cat feishu-chat://company/chats/研发群/2026-05-10/messages --range 0:20
mfs search "线上登录失败" feishu-chat://company
```

输出：

```text
[1] feishu-chat://company/chats/研发群/2026-05-10/threads  score=0.835
     thread: om_123
     线上登录失败从 10:42 开始，回滚后恢复。
```

## 5. 消息型 source 的默认行为

| 操作 | 行为 |
| --- | --- |
| `ls/tree` | 展示 workspace、channel、日期、附件 |
| `cat` / `cat --range` | 完整或区间展示消息 |
| `head/tail` | 查看消息 JSONL 的头尾 |
| `tail -f` | 跟随今天的频道或 thread |
| `grep` | 搜索消息文本，可使用 provider search |
| `search` | 按 thread、session 或 time-window 返回结果 |
| `export` | 把消息流写到本地，后续用 jq / Python 处理 |

## 6. MFS Index 配置重点

消息型 source 的搜索文本通常来自：

- thread title 或首条消息。
- message text。
- 附件抽取文本。
- 关键回复上下文。

metadata 通常是：

- channel。
- users。
- start_time / end_time。
- thread_ts。
- labels。

用户不需要先理解内部聚合，只需要知道搜索结果给出 thread/session 的上下文和可继续过滤的 URI。
