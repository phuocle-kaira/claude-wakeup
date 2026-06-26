# claude-wakeup

> **Closed (2026-06-26): the warmup idea does not work.** An automated daily "hi"
> cannot pre-anchor your interactive Claude usage window. On the Max plan the 5-hour
> "Current session" anchors only on a genuine *interactive* first message; automated
> runs (GitHub Actions `setup-token` heartbeats **and** native Claude Code cloud
> routines) are excluded and verified not to move it. Local anchoring is also out
> (machine is powered off overnight). The heartbeat workflow has been removed.
>
> Full evidence and reasoning:
> [`plans/260625-1428-claude-heartbeat-session-anchor/plan.md`](plans/260625-1428-claude-heartbeat-session-anchor/plan.md)
> (see *Outcome* and *Validation Log — Session 2*).
