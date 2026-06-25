---
phase: 2
title: "Activate and Verify"
status: pending
effort: "S"
---

# Phase 2: Activate and Verify

<!-- Updated: Validation Session 1 - anchor margin 05:07 -> 05:09; F2 mitigated via Phase 1 keep-alive -->

## Overview

User-owned activation: generate the OAuth token, set the repo secret, trigger a manual test run, and **verify in Settings → Usage** that the 5h window actually anchored. Enable failure alerting. This phase is the real success gate — it confirms the unverified mechanism premise.

Depends on Phase 1 (workflow file must be pushed first).

## Requirements

- Functional: secret `CLAUDE_CODE_OAUTH_TOKEN` set on the repo; manual `workflow_dispatch` run completes green; claude.ai Usage confirms window anchored ~05:00–05:30; GitHub failure email enabled.
- Non-functional: token never echoed, logged, committed, or pasted into any file. Generated and set by the user only.

## Architecture

`claude setup-token` mints an OAuth token bound to the user's subscription. Stored as a GitHub Actions secret, injected as env at run time. The heartbeat's `claude -p "hi"` call consumes a negligible slice of the subscription quota and (premise to verify) starts/touches the same rolling 5h window as interactive use.

## Related Code Files

- None (configuration + verification only; no repo file changes).

## Implementation Steps

1. Generate token locally (never shared): `claude setup-token` → copy printed token.
2. Set secret: `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo phuocle-kaira/claude-wakeup` (or UI: Settings → Secrets and variables → Actions → New repository secret, name `CLAUDE_CODE_OAUTH_TOKEN`).
3. Trigger test run: `gh workflow run heartbeat.yml` (or Actions tab → Run workflow).
4. Watch the run to green in the Actions tab.
5. **VERIFY (gate):** claude.ai → Settings → Usage — confirm the 5h window reset/anchored near the run time. If it did NOT move → the scheme is invalid; stop and reassess (do not rely on green alone).
6. Enable alerting: GitHub → Settings → Notifications → ensure email on Actions failure (expired token → job fails → email).
7. Optional: after a few real days, if drift/skips hurt, consider extra cron lines to pin more marks — only if needed (YAGNI).

## Success Criteria

- [x] Secret `CLAUDE_CODE_OAUTH_TOKEN` set (by user); token absent from all files/logs — proven: `Send heartbeat` step passed (run 28155728106, 2026-06-25)
- [x] Manual `workflow_dispatch` run completes green — run 28155728106, 13s, all 5 steps ✓; keep-alive commit `545cb36` pushed, no loop
- [ ] **Settings → Usage confirms the 5h window anchored** (decisive — STILL PENDING user check; manual run anchored ~15:00 GMT+7, not 05:09)
- [ ] 3 distinct sessions reachable across 09:30–12:00 and 13:15–17:45 on a normal day
- [ ] Failure email confirmed enabled

## Run log

- **2026-06-25 ~15:00 GMT+7** — manual `workflow_dispatch` run `28155728106` green in 13s. All steps passed incl. `Send heartbeat` (token valid) and `Keep-alive commit` (→ `545cb36`, `last-heartbeat.txt`=`2026-06-25T08:00:44Z`, no recursive trigger). Non-blocking annotation: `checkout@v4` forced to Node 24 (Node 20 deprecation). **Decisive Usage check not yet confirmed by user** — Open Question #1 remains open until claude.ai Usage is observed to move.
- **2026-06-25 (bump)** — `actions/checkout@v4` → `@v7` (commit `0fa0156`). Re-run `28155950258` green in 13s, **no annotations** (Node 20 deprecation cleared).

## Risk Assessment

- **Mechanism unverified (highest):** if Usage does not move, the whole plan is void. This step exists to catch that before trusting it.
- **Token expiry:** silent over time → mitigated by failure email (Step 6).
- **Cron skip/drift:** a skipped day falls back to 2 sessions; forward drift tolerated up to the ~07:00 lunch cliff (anchor 05:09 keeps ~1h50 margin).
- **60-day auto-disable (red-team F2):** **mitigated** by the keep-alive commit step in Phase 1. Residual: bot-commit activity isn't guaranteed to reset the timer — keep a ~50-day awareness check; if the morning anchor vanishes (a disable raises no failure email), re-arm by pushing any commit or "Enable workflow" in the Actions tab.
- **First-run trust/interactive prompt:** `-p` print mode with no tools should not prompt; `timeout-minutes` (Phase 1) bounds any hang.
- **Rollback:** remove the secret and/or disable/delete the workflow; no data or repo state to revert.
