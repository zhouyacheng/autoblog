---
title: "Kanban Swarm 拆解：Hermes Agent 为什么不再写第二个调度器"
date: 2026-10-27T09:22:57+08:00
tags: [hermes, kanban, multi-agent, orchestration]
author: zyc
---

我照官方文档敲了那条 swarm 示例，第一秒就失败了：

```bash
$ hermes kanban swarm "Design a multi-region failover plan" \
    --workers researcher,architect,sre \
    --verifier reviewer --synthesizer writer
error: unrecognized arguments: --workers researcher,architect,sre
```

`website/docs/user-guide/features/kanban.md:1065-1067` 写的是 `--workers`，而 argparse 里只注册了可重复的 `--worker PROFILE:TITLE[:SKILL,SKILL]`（`kanban_parser.py:224`）。这个笔误反而是个好切口——只有能一行行读完的实现，才谈得上判断它到底解决了什么。`hermes_cli/kanban_swarm.py` 全文 300 行。

## 一条命令写出的四层卡图

```text
planning root (建完立即 done)
   ├─ worker × N   parents=[root]
   └─ verifier     parents=所有 worker
        └─ synthesizer   parents=[verifier]
```

root 用 `initial_status="blocked"` 创建，再在同一事务内翻成 `done`，root 的 body 里写明理由：「completed immediately so parallel workers can start while it remains the shared blackboard and audit anchor」。verifier 自动带 `skills=["requesting-code-review"]`，synthesizer 自动带 `skills=["humanizer"]`——硬编码在源码里，不是配置项。

worker 卡也不是裸的 title。`_swarm_context()` 往 worker、verifier、synthesizer 三者的 body 尾部各追加一段 `## Swarm protocol`：共享黑板的 root id、先读 sibling/parent handoff、把机器可读事实写进 completion metadata、把跨 worker 的笔记写回 root 卡的结构化评论。为什么重要：这些 worker 是 dispatcher 拉起来的独立进程，彼此没有内存里的共享上下文，它们之所以知道 root 在哪，全靠这段注入文本。

依赖边也不止一条。`create_task(parents=[...])` 对每个 parent 往 `task_links` 插一行记录（`INSERT OR IGNORE`，`kanban_db.py:1345`），N 个 worker 就是 **2N+1** 条：root→worker 共 N、worker→verifier 共 N、verifier→synthesizer 1 条。我建 2 个 worker 后实测 `SELECT count(*) FROM task_links` = 5。2N+1 是「verifier 必须等全部 worker」这句话在数据层的全部含义——没有 fan-in 语法糖，只有一张 verifier 卡挂着 N 个 parent，ready 判定交给既有的依赖机制。

## 第二个调度器为什么不写

建库后 `sqlite_master` 里是 8 张表：`tasks`、`task_links`、`task_comments`、`task_events`、`task_runs`、`task_attachments`、`kanban_notify_subs`、`sqlite_sequence`。swarm 没有新表、没有新迁移，状态全落在 `tasks` / `task_links` / `task_comments` 这三张既有的表里。这 300 行里也没有 gateway、没有 notifier、没有 dashboard 路由——因为那些组件读的就是同一套表，它们本来就在跑，绕不过去。

更要紧的是 swarm 不 spawn 任何进程，它只写库。真正拉起 worker 的是 `kanban_db_dispatch` 里的 dispatcher，并发由 `kanban.max_in_progress` 与 `kanban.max_in_progress_per_profile` 控制（默认均为 `None`，此时走推导出的默认值）。所以「swarm 让 N 个 worker 并行」这种说法是错的——swarm 只保证 N 张卡同时 ready。想要更并行，改 swarm 参数没有用。

对照同类项目，取舍就看得很清楚。LangGraph 的状态层要使用者自己接：`compile(checkpointer=...)`、自己传 `thread_id`、自己在 `SqliteSaver`/`PostgresSaver` 之间选，durability 还要在三档里显式挑。CrewAI 的 `Crew(agents, tasks, process)` 自带执行器（`sequential`/`hierarchical`）。两者都必须回答「状态放哪、谁负责推进」，Hermes 的答案是：`kanban.db` 已经在跑了，别再答第二遍。

回报是可验证的：崩溃后 `hermes kanban list` 读到的是同一张图，dashboard、notifier、slash command、`kanban_*` toolset 全部免费复用，因为它们从来没被绕过。为什么重要：多 agent 编排的难点从来不是画图，是状态放哪——状态层选错，你就得为它再写一个调度器，然后为这个调度器再补一套审计和通知。

## 原子性与幂等：一个事务、一次 CAS、一条黑板记录

`create_swarm` 把整张图包在一个 `write_txn` 里，root 的 `blocked → done` 走 CAS，没有调用 `kb.complete_task`。注释给了原因：那会带来「own transaction + post-commit side effects that would run while the outer txn can still roll back」。提交之后才调用 `recompute_ready(conn)`、再触发 `kanban_task_completed` hook，而 `recompute_ready` 的 docstring 明确要求「call OUTSIDE any write txn」。CAS 之后 `rowcount != 1` 时，抛 `RuntimeError` 的是外层 `create_swarm`，不是那个 helper——helper 只 `return False`。

`tests/hermes_cli/test_kanban_swarm.py` 5 个用例本机跑过：`5 passed in 1.58s`。其中一个用第二个 connection 当 reader，断言外层事务未提交时 `COUNT(*) FROM tasks` 为 0，以及 hook 触发时 `writer.in_transaction is False`。为什么重要：只要 hook 在外层事务里跑，读者就可能看到一张随后被回滚的图；这个断言就是「要么看不见，要么看全」的具体形式。

幂等的支点是 root 卡上一条评论（前缀 `[swarm:blackboard]` 后面紧跟一个空格）：

```text
[swarm:blackboard] {"key": "topology", "value": {
  "goal": "把调研稿变成可发布文章",
  "root_id": "t_65e04e4f",
  "worker_ids": ["t_e55d18f5", "t_07a2192f"],
  "verifier_id": "t_11789979",
  "synthesizer_id": "t_03805c60"
}}
```

`idempotency_key` 命中已有 root 时，从 `latest_blackboard().get("topology")` 恢复 id 后直接返回，不重复建图。实测同一个 key 建两次：`tasks` 仍是 5 行，root 与 worker id 完全一致。这条性质比并行更值钱——卡被 reclaim、失败重跑、人工重试在多 agent 流水线里是常态，能安全重放而不长出第二张图，才敢放进生产。

写入容易，读取有坑：`latest_blackboard()` 逐条扫 root 卡评论，只认以 `[swarm:blackboard]` 加一个空格开头、JSON 可解析、`key` 为非空字符串的条目；同一个 key 后写的覆盖先写的，胜出者的作者汇总进 `_authors`。三类坏输入全部静默跳过，没有 warning，也没有 event。为什么重要：worker 想用黑板传结构化事实，一个尾随逗号就能让更新彻底消失，而且没人会告诉你。

`idempotency_key` 在 schema 上还只是普通索引 `idx_tasks_idempotency`，不是 UNIQUE 约束。去重靠「先 `SELECT id ... ORDER BY created_at DESC LIMIT 1`、再把两步包进一个事务」。知道这点才知道幂等的边界：它是先查后写，并发下兜底的是事务串行化，不是约束。

## verifier 的 gate 是纪律，不是平台强制

verifier 的 body 写着「complete only with metadata {"gate": "pass"}… otherwise block with exact missing work」。但全仓 grep `gate` 只命中两处，都在 `kanban_swarm.py` 的 prompt 字符串里；调度侧没有任何代码读 `metadata.gate`。

synthesizer 能不能 ready，只看 verifier 卡的 `status` 是否变成 `done`。也就是说 verifier 只要 complete——哪怕不带 gate、不带证据——synthesizer 照样醒。

为什么重要：把 gate 当成强制闸门去设计流水线，会在 verifier 第一次偷懒时静默失效——没有报错，也没有告警。要真闸门，得自己补校验：读 `metadata.gate`，不是 `pass` 就打回，或者干脆不让 verifier 走 `kanban_complete`。另外 `topology` 那条记录的作者取自 `created_by`（默认 `swarm-orchestrator`），不是某个 worker，别指望从 `_authors` 里看出 worker 写了什么。

## 结论：按形状选工具

修好参数后的真实命令（parser 实测可解析，`--worker` 重复即可）：

```bash
hermes kanban swarm "把调研稿变成可发布文章" \
    --worker "researcher:调研:arxiv" \
    --worker "architect:提纲" \
    --verifier reviewer --synthesizer writer \
    --idempotency-key blog-2026-10-27
```

- **fan-out → 收敛**：用 swarm。N 路并行调研加一份汇总，就是它的形状。
- **严格串行、带「发布」位**：用 `hermes kanban create --parent`。`--parent` 可重复（`kanban_parser.py:168`），链式表达力强于固定四层图，发布位不该塞进 synthesizer。
- **要硬 gate**：别只信 prompt，自己写校验。
- **想调并行度**：改 `kanban.max_in_progress`，不是 swarm 参数。
- **要重跑**：带同一个 `--idempotency-key`，命令就会从黑板恢复原图，而不是再画一张。

版本基线 `v2026.9.24 (git.9b904dc)`。本机 clone 是 shallow（`git log` 仅 1 个 commit），swarm 是从哪一版引入的，无法核实（未证实）。另需更正一个常见说法：英文文档已有 swarm 小节，中文站与 tutorial 页搜 `swarm` 命中 0 条——不是「文档没写」，是翻译没跟上，而那段英文示例本身就是错的。
