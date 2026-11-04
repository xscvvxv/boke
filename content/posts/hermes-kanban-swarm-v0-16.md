---
title: "v0.16 没有引入 Kanban Swarm：Hermes 多智能体编排的机制与边界"
date: 2026-11-04T20:45:00+08:00
tags: ["hermes", "kanban", "multi-agent", "sqlite"]
author: "xscvvxv"
---

「Hermes Agent v0.16 引入了 Kanban swarm」——这句话在二手资料里流传很广，但它是错的。`hermes kanban swarm` 首发于 v0.15.0 Velocity（2026-05-28），比 v0.16.0 Surface 早八天。它的价值也不在并行，而在把多智能体的状态从 prompt 搬进了 SQLite。

<!--more-->

## 版本：一个 commit 就判完了

功能 commit 是 `3ee7a554`（2026-05-18），标题 `feat(cli): add kanban swarm topology helper`。祖先关系三步判定：它不是 v0.14.0（`a91a57f`）的祖先，是 v0.15.0（`0c859a1`）的祖先，也是 v0.16.0（`3c231eb`）的祖先。在 v0.16 里能查到，那是继承，不是引入。v0.15.0 的 release commit 原文写着「kanban grows into a multi-agent platform (104 PRs)」；v0.16.0 的原文是桌面端、浏览器管理面板、远程 gateway、中文 UI、`/undo`，通篇没有 kanban 和 swarm。

二手资料之所以张冠李戴，是因为 v0.16 把看板做成了可视化界面，swarm 第一次「看得见」，于是被当成「第一次存在」。另有一个坑：v0.15、v0.16 的 GitHub release tag 页现在返回 404，没法引 URL。要自己验证，就 `git log --diff-filter=A -- hermes_cli/kanban_swarm.py`，或者看被 salvage 的 PR #26791。

## 没有第二个调度器，可并行是真的

源码 docstring 第一句：「Deliberately no second scheduler — a small task graph written into the existing Kanban kernel」。swarm 只做一件事：往既有看板里写一张图。root 卡建完立刻完成；每个 worker 卡的 parent 都是 root，所以彼此并行；verifier 的 parents 是全部 worker，构成 fan-in 屏障；synthesizer 的 parent 只有 verifier，严格串行。拓扑不是 CLI 画出来的，是 `task_links` 里的 parent 边决定的。

命令长这样（本机 `hermes kanban swarm --help` 的签名，`--worker` 可重复）：

```bash
hermes kanban swarm \
  "给 blog 仓库补一篇 Kanban swarm 的机制说明" \
  --worker researcher:"调研：版本归属与失败语义" \
  --worker writer:"初稿：机制与边界" \
  --worker writer:"附录：可复现命令" \
  --verifier reviewer \
  --synthesizer writer \
  --tenant blog \
  --priority 5 \
  --idempotency-key kanban-swarm-post
```

`--worker` 写几条就多几张并行卡，`--verifier` 和 `--synthesizer` 是必填项，没有默认值。上面这条跑完，board 上多出 N+3 张卡：1 张 root（建完立刻 `done`，只当黑板用）、3 张 worker、1 张 verifier、1 张 synthesizer，正好 6 张。依赖边 7 条：root 指向每个 worker，每个 worker 指向 verifier，verifier 指向 synthesizer。图是原子提交的——要么一张都看不见，要么整张齐全。

两个细节值得记。一是同一个 profile 可以挂多张卡：上面两张 worker 都归 `writer`，卡是任务，profile 只是执行身份。二是 `--idempotency-key` 的语义在 `--help` 里写得很直白——`Dedup key for the root card`：同一个 key 重跑一次，不会再建一张重复的 root。`--tenant`、`--priority`、`--idempotency-key` 这三个参数摆在一起，说明它的定位不是「多 agent 演示」，而是能进流水线的建图工具：重跑不重卡、租户不串线、优先级能压过别的卡。

之后的事全归原来的 dispatcher：卡在 `todo` 等所有 parent 变 `done`，被提升为 `ready`，每个 tick 原子 claim 一次，写一行 `task_runs`、把 `tasks.current_run_id` 指过去，进 `running`；worker 每次工具调用都顺手把存活状态镜像回看板，大约每分钟一次。没有新服务，没有新队列，看板上的东西和人手动建卡完全同构。

「真并行」的判据不是文档说并行，而是三条结构性事实：每个 worker 是独立 OS 进程（`task_runs.worker_pid` 逐 run 记录）、独立 session、独立 workspace。所以 N 个 worker 的墙钟接近 max 而不是 sum，前提是它们之间不互相等待。

verifier 为什么不放在最后？它校验的是每个 worker 的结构化 handoff，不是成稿。等合并完才发现问题，纠正成本就从「驳回一张卡」变成「重写一节」。

并发上限这里文档和实现对不上：文档写 `--workers a,b,c`，实测 CLI 是 `--worker PROFILE:TITLE` 可重复；文档说 `max_in_progress` 不设即无限，源码在能读到内存时推导 `clamp(MemTotal/512MB, 2, 8)`，只有 macOS、Windows 才返回不限。照文档抄命令，第一行就报错。

## 状态全在 SQLite：八张表，零条 prompt

worker 之间共享的东西一件都不在 prompt 里。黑板是 root 卡上 `[swarm:blackboard] ` 前缀的结构化 JSON 评论；单个 worker 的产出是 `task_runs.summary` 加 `metadata` 两列；下游 worker 启动时调 `kanban_show()`，返回体已经预格式化好 `worker_context`，包含标题、正文、父卡 handoff、历史尝试和完整评论线程。全部状态躺在八张表里：`tasks`、`task_runs`、`task_events`、`task_comments`、`task_attachments`、`task_links`、`kanban_notify_subs`，外加 SQLite 自带的 `sqlite_sequence`。

这么放不是为了好看。状态落在既有表里，看板、通知器、slash 命令和 dispatcher 一行都不用改就能继续工作。更关键的是它顺手治好了上下文爆炸：没有哪个 agent 需要把别人的全文读进窗口，它只读自己那一份切片。

## 边界：失败怎么记账，什么时候别用

失败有四条处理路径，各自配一组常量，账算得不一样。stale 回收要求跑满 4 小时且 1 小时无心跳，回 `ready` 但不计失败——属于缺席检测，锅不算在卡上。claim-TTL 过期的自动回收记 1 次非成功，否则会无限 claim→reclaim 循环。手动 reclaim 清零。连续 2 次非成功就 `gave_up` 自动 block，同因反复 block 到 2 次转 `triage`。所以「reclaim 等于免费重试」只对 stale 和手动 reclaim 成立，把回收当无限重试，第三次会发现卡已经被 block 了。

这不是纸面推演。今天这篇文章就是同一套内核跑的，只不过四张卡串成一条 parent 链，不是上面那张扇出图。时间戳能逐条查：`kanban.db` 的 `task_events` 里，20:04:07 建 root 卡，10 秒后被 claim。第一轮跑了 14 分钟才被人手动 reclaim，回收理由原文是「worker 卡在 computer_use/tool_search，12 分钟无进展」——一次 `tool_search` 走了 420 秒超时才失败，紧接着它又去调 `computer_use`，被审批拦住，剩下的时间都花在跟这篇稿子无关的工具上。20:18:56 改掉卡上的指令（明确禁用无关工具），20:19:04 重跑，20:20:48 `done`，1 分 44 秒。后面三跳：researcher 15 分 57 秒，writer 7 分 35 秒，reviewer 4 分 38 秒。

于是可以说准一点：reclaim 是免费重启，不是免费时间。那次回收没给卡记失败，但 14 分钟已经花掉了，没人退得回来。真正把时间省下来的是改指令那一下——把「准用什么工具」写清楚，比多给一次重试有用得多。卡住时该往卡里写什么，答案也就在这里：不是「请继续」，而是「上一轮卡在哪一步、下一步只准做什么、哪些工具别再碰」。另有一笔容易被忽略的开销在相邻两跳之间：root 卡 20:20:48 完成，researcher 20:21:12 才被 claim；writer 那一跳等了 56 秒，reviewer 等了 57 秒。dispatcher 按 tick 扫，每跳都有几十秒排队，三跳加起来两分多钟。这段延迟跟任务难度无关，只跟图上有几跳有关。

成本也是账。token 可加：每个 worker 独立 session，没有窗口共享。墙钟在真并行下取 max。固定开销是 N 次冷启动加 verifier、synthesizer 两跳。

反例不需要编。三个文件、两百行的重构套四个 worker，负收益长这样：单 agent 一次读入、一次编辑、一次跑测试就收工；四个 worker 各自把同样的三个文件读进各自窗口、各自冷启动、各自跑一遍测试才敢 claim 完成，verifier 得把四份 handoff 读全，synthesizer 再把四份改动合成一个两百行的 diff，最后总得有人跑一次全量测试确认合并没有打架。省下的是编辑时间，赔进去的是四份上下文重建、一次合并、一次校验——在两百行这个量级上，后者一定更贵。协调成本吃掉它省下的时间，并行这时是负收益。

形状本身也是固定的：swarm v1 就是 1 root 加 N workers 加 1 verifier 加 1 synthesizer，没有动态拓扑。它设计上还是单机的——状态在本地 SQLite，dispatcher 只在本机 spawn，崩溃检测假设 PID 是本机的，跨主机明确不支持。要跨机器编排，这套东西现在不是答案。

判断清单，命中任意两条就别上：一个人一个窗口能干完；worker 之间强顺序耦合，该用 parents 串行链；问题还没定义清楚，先单人 spike；每步都要人肉审批或有外部副作用；只需要多跑几个视角、不需要可恢复性，`delegate_task` 更便宜；说不清「做完」长什么样。

反过来，四条全满足才值得上：子任务真正独立、工作量明显大于固定开销、需要跨会话恢复或事后审计、有可验证的完成句柄（测试通过、PR 链接、CI 变绿）。缺最后一条，verifier 会退化成主观打分。

参考：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban