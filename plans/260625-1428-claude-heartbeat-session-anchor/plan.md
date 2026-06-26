---
title: "Claude Heartbeat Session Anchor Workflow"
description: "Daily GitHub Actions heartbeat anchoring the Claude 5h window at ~05:09 GMT+7 to yield 3 usage sessions across a split workday"
status: closed-infeasible
priority: P2
branch: "main"
tags: [github-actions, claude-code, automation]
blockedBy: []
blocks: []
created: "2026-06-25T07:31:32.379Z"
createdBy: "ck:plan"
source: skill
---

# Claude Heartbeat Session Anchor Workflow

## Outcome — CLOSED, scheme infeasible (2026-06-26)

The make-or-break premise (Open question #1) was **empirically disproven**. On 2026-06-26 two automated warmups both ran successfully — the GitHub Actions `claude -p "hi"` at 06:38 GMT+7 (run `28207382111`, real reply `Hi! How can I help you today?`) and a separately-existing native Claude Code cloud routine `daily-work-heartbeat` at 05:31 GMT+7 — yet **Settings → Usage still showed the interactive "Current session" anchored at 09:30** (reset ~14:30), i.e. on the user's own first message.

Root cause: on the Max plan there is **one shared interactive 5h "Current session" window**, and it anchors only on a genuine **interactive** first message. **Automated runs (CI `setup-token` heartbeats AND native cloud routines) do not anchor or count toward it.** No automated, machine-independent method can shift the interactive reset boundary. Local anchoring is also ruled out (Mac powered off overnight). Therefore the goal (pre-anchor S1 while asleep → 3 sessions) is **not achievable** as designed.

Actions taken on closure: native routine `daily-work-heartbeat` paused; GitHub Actions heartbeat workflow removed. See Validation Log Session 2.

## Overview

Add a scheduled GitHub Actions workflow that runs `claude -p "hi"` once daily at **05:09 GMT+7** to anchor the Claude subscription's rolling 5-hour usage window early (while the user's Mac is asleep). Anchoring early pushes the session reset boundaries into work hours, yielding **3 usage sessions** across the split workday (09:30–12:00 + 13:15–17:45) instead of 2. The workflow also self-commits a keep-alive timestamp to dodge GitHub's 60-day auto-disable.

Source brainstorm + timing verification: [`../reports/brainstorm-260625-1402-claude-heartbeat-window-anchor-report.md`](../reports/brainstorm-260625-1402-claude-heartbeat-window-anchor-report.md)

## Goal & acceptance

- **Goal:** 3 distinct 5h sessions reachable during the workday via early anchor.
- **Acceptance (the only real check):** claude.ai → Settings → Usage shows the 5h window anchored ~05:00–05:30 after a real run. A green workflow alone does NOT prove success.

## Key decisions (from brainstorm, locked)

- Platform: **GitHub Actions** (Mac asleep at anchor time → cloud required).
- Anchor: **`cron: 09 22 * * *`** = 05:09 GMT+7. Off-peak minute (5:07/5:09/5:11 are equivalent; avoid multiples of 5 like 5:10 which are busier).
- **Hard ceiling: anchor must not exceed ~06:00.** At ≥07:00 the S1 reset lands in the lunch break (12:00–13:15), the reset-on-next-message grid shifts, and sessions drop to 2. GitHub cron only drifts forward → earlier anchor = safer.
- YAML is hardened: heartbeat + `timeout-minutes: 10` + `concurrency` + `permissions: contents: write` + checkout + keep-alive commit.
- **Keep-alive commit** (validation decision): each run writes/commits `.github/last-heartbeat.txt` to keep the repo "active" and dodge the 60-day auto-disable.
- **Alerting:** GitHub default failure email (no extra code/secret).
- Single heartbeat only (controls S1; S2/S3 re-anchor on the user's own messages — fine, user is active then).

## Alternatives considered

- **Local `launchd` + `pmset` auto-wake** (per a Reddit r/ClaudeCode post): wake the Mac at 05:00 via `pmset repeat wake`, run local `claude` via launchd. Advantages: no cloud token/secret, precise timing, no 60-day disable. **Rejected:** the user's Mac is **shut down / unplugged overnight**, and `pmset` cannot wake a powered-off machine. GitHub Actions is chosen for machine-independence.
- **`vdsmon/claude-warmup`** (GitHub repo): identical GitHub-Actions + `claude setup-token` approach, in production use — confirms this plan's method is standard. This plan is a hardened superset (off-peak minute, lunch-cliff anchor, timeout/concurrency, keep-alive).

## Phases

| Phase | Name | Status |
|-------|------|--------|
| 1 | [Provision Workflow](./phase-01-provision-workflow.md) | Done (`e23daeb`, 2026-06-25), then **reverted** on closure (2026-06-26) |
| 2 | [Activate and Verify](./phase-02-activate-and-verify.md) | **Failed** — verification disproved the premise (2026-06-26) |

Phase 2 depends on Phase 1.

> **Status 2026-06-25 (cook):** Phase 1 shipped — `claude-heartbeat` workflow live on `main`, active in Actions tab, 0 runs (awaiting secret). Phase 2 is user-owned per Hard rules (mint token, set `CLAUDE_CODE_OAUTH_TOKEN` secret, dispatch a test run, verify Settings → Usage). Plan stays `in-progress` until the Usage check confirms the mechanism premise (Open question #1).

## Hard rules (non-negotiable)

- Never read, echo, log, or store the OAuth token — must never appear in any file or command.
- Do NOT set the GitHub secret on the user's behalf — user owns that step.
- Do not install the GitHub App. Change nothing else in the repo.

## Risk summary

- **Unverified mechanism (highest):** CI-token heartbeat may not move the interactive usage window. Gate on Phase 2 Usage check.
- **Cron unreliability:** drift 5–30min, occasional skipped day (→ 2 sessions that day). Email-on-fail covers hard failures, not silent skips.
- **60-day auto-disable (red-team F2):** GitHub disables scheduled workflows after ~60 days of repo inactivity. **Mitigated** by the keep-alive commit step (validation decision). Residual: bot (`GITHUB_TOKEN`) commits *should* count as activity but aren't guaranteed — keep a ~50-day awareness check; if the morning anchor disappears, re-arm manually.
- **Lunch-break cliff:** keep anchor ≤ 05:30 in practice; never exceed 06:00.
- **Token expiry:** eventual `CLAUDE_CODE_OAUTH_TOKEN` expiry → job fails → GitHub email.
- **Non-issue (verified):** no DST drift — GMT+7 and UTC both have no DST, so `22:09 UTC` = `05:09 GMT+7` year-round.

## Red-team verdict

Inline adversarial review run (hard mode). No blocking findings; design survives. Key residual: the mechanism premise is empirical and must be confirmed in Phase 2 before trusting the scheme. Full notes: [`./red-team-heartbeat-anchor-review.md`](./red-team-heartbeat-anchor-review.md).

## Dependencies

None (no other plans; fresh repo with only `README.md`).

## Open questions

1. Does a GitHub Actions (CI-token) heartbeat actually move the interactive usage window on this account? → **RESOLVED: NO** (2026-06-26). Neither the CI heartbeat nor a native cloud routine anchored the interactive Current session; only the user's own first interactive message does. Premise dead → scheme closed.
2. Does a `GITHUB_TOKEN` keep-alive commit actually reset GitHub's 60-day timer? → **uncertain**; monitor at ~50 days, manual re-arm as fallback.

_(Resolved in validation: anchor = 05:09; alerting = email-only.)_

## Validation Log

### Session 1 — 2026-06-25 (mode: prompt, Light tier)

**Verification Results**
- Claims checked: 5 · Verified: 5 · Failed: 0 · Unverified: 0 · Tier: Light (2 phases)
- branch `main` ✓ · remote `phuocle-kaira/claude-wakeup` ✓ · `heartbeat.yml` absent ✓ · cron `09 22 UTC` = 05:09 GMT+7 ✓ · `gh` authenticated ✓
- Note: working tree "dirty" only from new `plans/` artifacts (expected); Phase 1 uses scoped `git add`.

**Decisions confirmed**
- Q1 (F2 mitigation): **Add keep-alive commit step** → Phase 1 updated with checkout + `permissions: contents: write` + timestamp commit.
- Q2 (anchor): user floated 5:09/5:11 → **05:09 (`09 22 * * *`)** chosen; 5:07/5:09/5:11 equivalent, avoid multiples of 5.
- Q3 (alerting): **GitHub default failure email** (no extra code/secret).

**Phase propagation**
- Phase 1: anchor 05:07→05:09, keep-alive step added, risks updated. Marker added.
- Phase 2: anchor-margin reference 05:07→05:09; F2 mitigation note updated to reflect keep-alive.

### Session 2 — 2026-06-26 (empirical verification → FAIL, plan closed)

**Setup at test time:** Both warmups live and successful the same morning.
- GitHub Actions run `28207382111`: scheduled, success, 06:38 GMT+7 (cron 05:09 + ~1h29 GitHub drift). Log line 129 shows real model reply `Hi! How can I help you today?` → token valid, message genuinely sent.
- Native Claude Code cloud routine `daily-work-heartbeat` (pre-existing, under Phuoc·Max account): ran today 05:31 GMT+7, success; ~1 week of green daily runs.

**Observation (decisive):** Settings → Usage at ~10:30 GMT+7 — **Current session: "Resets in 3 hr 53 min" (≈14:30) = anchored 09:30** (user's first interactive message). Weekly "All models" resets Thu 09:00. So neither 05:31 nor 06:38 automated warmup touched the interactive Current session.

**Conclusion:** Premise false. Max plan = one shared interactive 5h window; anchors only on a genuine interactive first message; automated runs (CI setup-token + native routine) are excluded. Goal infeasible by any machine-independent method (local ruled out — Mac off overnight).

**Cross-repo check:** `vdsmon/claude-warmup` (GitHub Actions + setup-token) and `tappress/claude-code-warmup` (Vercel cron + Anthropic API, same setup-token) both use the identical mechanism this plan used → same failure. vdsmon README (2026-04-01) points to native Claude Code Web scheduled tasks as the "native" path — which is exactly the `daily-work-heartbeat` routine that also failed here.

**Closure actions:** routine paused (toggle → Paused); GitHub Actions heartbeat workflow + `last-heartbeat.txt` removed; plan status → `closed-infeasible`.

### Whole-Plan Consistency Sweep
- Re-read `plan.md`, `phase-01`, `phase-02`, red-team review. Operative anchor is now `05:09`/`09 22` everywhere it drives behavior (Overview, Key decisions, Phase 1 YAML, Phase 2 margin, DST example, description). F2 described consistently as "mitigated by keep-alive, residual uncertainty".
- Intentional remaining `05:07` mentions (non-contradictory): (a) `red-team-heartbeat-anchor-review.md` uses 05:07 as its representative anchor — its conclusions (DST stable, quota negligible, 3 sessions across the 05:07–06:30 range) all hold for 05:09; (b) the Key-decisions/Validation-Log lines that cite "5:07/5:09/5:11 equivalent" by design; (c) the brainstorm report outside the plan dir. None drive implementation.
- Unresolved contradictions: **none**.
