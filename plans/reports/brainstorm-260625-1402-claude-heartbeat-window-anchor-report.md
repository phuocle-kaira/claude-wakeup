# Brainstorm — Claude 5h-Window Anchor via Daily Heartbeat

- Date: 2026-06-25 (Asia/Saigon, GMT+7) · Revised 14:28 (schedule refined + English)
- Skill: `/brainstorm` · Flags: none · Mode: report-only → handoff to `/ck:plan --hard`
- Repo: `phuocle-kaira/claude-wakeup` · Branch `main` · tree clean
- Status: **Design approved, not implemented** (user will own the secret + push)

---

## 1. Objective & requirements

Maximize the number of distinct **5-hour usage sessions** reachable during the workday by anchoring the first session early (while user asleep) with an automated `claude -p "hi"` heartbeat. Anchoring early pushes the session reset boundaries *into* work hours, yielding **3 sessions instead of 2**.

| Item | Decision |
|---|---|
| Goal | Most sessions possible during work hours — **target = 3** (the ceiling) |
| Workday | **Morning 09:30–12:00** · lunch 12:00–13:15 · **Afternoon 13:15–17:45** (7h work, 8h15 span) |
| Output | `.github/workflows/heartbeat.yml` running `claude -p "hi"` daily ~05:07 GMT+7 |
| Acceptance | claude.ai → Settings → Usage confirms the 5h window anchors ~05:00–05:30 |
| Platform | GitHub Actions (user's Mac is asleep at anchor time → cloud required) |
| Anchor target | **05:07 GMT+7** (`cron: 07 22 * * *`). Hard ceiling: do NOT exceed ~06:00 |
| Alert on fail | Yes — GitHub default email on scheduled-workflow failure |
| Out of scope | Pinning all 4 daily marks; push notifications (Telegram/Slack); setting the secret; installing GitHub App |
| Hard rules | Never read/echo/store OAuth token · never set the secret on user's behalf · no GitHub App · change nothing else in repo |

---

## 2. Mechanic (the model everything rests on)

- Usage limit is a **rolling 5h window**. The window **starts on the first message** and expires 5h later.
- After expiry the **next window does not auto-start** — it starts on the **next message** (reset-on-next-message).
- Implication: a heartbeat only controls **Session 1's** anchor. Sessions 2 and 3 re-anchor on the user's own first message after each expiry — fine, because the user is actively working at those moments.
- ⚠️ **Unverified premise:** that a heartbeat sent from CI (via `CLAUDE_CODE_OAUTH_TOKEN`) moves the *same* window as interactive usage. Must be confirmed empirically (Settings → Usage) before trusting the scheme.

---

## 3. Core analysis — the 3-session strategy + the lunch-break cliff

**Why the heartbeat helps.** Without it, the user's first message at 09:30 anchors Session 1 → boundaries at 14:30 / 19:30 → only **2 sessions** in the workday. The heartbeat "pre-burns" Session 1 while the user sleeps, moving boundaries to ~10:00 / ~15:00 — both inside work hours → **3 sessions**.

**The lunch break creates a hard ceiling.** To get 3 sessions, Session 1's reset (`anchor + 5h`) must land **before lunch (12:00)** so Session 2 re-anchors mid-morning. If that reset lands **in lunch (12:00–13:15)**, Session 1 expires while the user is away; Session 2 only starts at 13:15, the grid shifts, and the 3rd boundary falls after work → drops to **2 sessions**.

- Reset of S1 = `anchor + 5h`. Need it **before ~11:30** (comfortably mid-morning, user actively messaging).
- `anchor + 5h ≥ 12:00` ⇒ `anchor ≥ 07:00` ⇒ **cliff → 2 sessions**.

**GitHub cron only drifts forward (later), never earlier** — so drift always pushes the anchor *toward* the 07:00 cliff. Earlier target = larger safety margin. Therefore:

- **05:07 target** → even +1h50 drift stays < 07:00. Safe.
- **06:00 target** → +1h drift = 07:00 = cliff. Fragile.

There is **no upside** to a later anchor: it does not add sessions (3 is the ceiling either way) — it only shifts coverage from S3 to S1 (total work coverage is fixed at 7h) while reducing drift safety.

---

## 4. Verification (simulation)

Simulated reset-on-next-message against the split schedule (dense messaging during work). Sessions = distinct 5h windows touched during work hours.

| Anchor | S1 reset | Sessions in workday | Verdict |
|---|---|---|---|
| no heartbeat (organic 09:30) | 14:30 | **2** | baseline — what we beat |
| **05:07 (recommended)** | 10:07 | **3** | ✅ ~1h53 margin to cliff |
| 05:30 | 10:30 | **3** | ✅ ~1h30 margin |
| 06:00 | 11:00 | **3** | ⚠️ on-time only, ~1h margin |
| 06:30 | 11:30 | **3** | ⚠️ thin margin |
| 07:00 | 12:00 (lunch) | **2** | ❌ cliff — S1 reset in lunch, grid resets |

Recommended timeline (**anchor 05:07**):

| Session | Window | Work coverage |
|---|---|---|
| S1 | 05:07–10:07 (heartbeat) | 09:30–10:07 (37m) |
| S2 | 10:07–15:07 (first msg after 10:07) | 10:07–12:00 + 13:15–15:07 (3h45, straddles lunch) |
| S3 | 15:07–20:07 (first msg after 15:07) | 15:07–17:45 (2h38) |

→ **3 sessions**, full workday covered. ✅

---

## 5. Final design — workflow

GitHub Actions, single heartbeat at **05:07 GMT+7**, off-peak minute (lower GitHub delay/skip rate than `:00`/`:30`). Hardened = user's original YAML + `timeout-minutes` + `concurrency`. Anchor logic unchanged; token only referenced via `secrets.*`.

```yaml
name: claude-heartbeat

on:
  schedule:
    - cron: "07 22 * * *"   # 22:07 UTC = 05:07 GMT+7 (next day) — off-peak minute
  workflow_dispatch:

concurrency:                 # avoid overlap if a manual dispatch coincides with the cron run
  group: claude-heartbeat
  cancel-in-progress: false

jobs:
  heartbeat:
    runs-on: ubuntu-latest
    timeout-minutes: 10      # prevent silent hang if claude waits for input
    steps:
      - name: Install Claude Code
        run: npm install -g @anthropic-ai/claude-code
      - name: Send heartbeat
        env:
          CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
        run: claude -p "hi"
```

> To keep the user's original verbatim YAML: drop the `concurrency` block and `timeout-minutes`, and revert cron to `30 22 * * *` (05:30, still 3 sessions, less drift margin).

---

## 6. Self-serve implementation checklist (user owns these)

1. Generate OAuth token locally (never shared): `claude setup-token` → copy printed token.
2. Add repo secret: `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo phuocle-kaira/claude-wakeup` (or UI: Settings → Secrets and variables → Actions → New repository secret, name `CLAUDE_CODE_OAUTH_TOKEN`). Token must not appear in any file.
3. Create `.github/workflows/heartbeat.yml` with §5 content.
4. Commit `Add daily Claude heartbeat workflow`, push to `main`.
5. Test: `gh workflow run heartbeat.yml` → watch Actions tab until green.
6. **VERIFY (the only real success check):** claude.ai → Settings → Usage — confirm the 5h window reset/anchored near the run time. A green workflow alone does NOT prove success.
7. Enable alerting: GitHub Settings → Notifications → ensure email on Actions failure (expired token → job fails → email).

---

## 7. Risks

- **Unverified mechanism (highest):** if CI heartbeat does not move the interactive window, the whole scheme is void. Gate on step 6.
- **Cron unreliability:** GitHub scheduled runs drift 5–30min, occasionally skip a day. A skipped day falls back to 2 sessions (organic 09:30 anchor). Email-on-fail covers hard failures, not silent skips.
- **Lunch-break cliff:** anchor must stay well below 07:00. Keep target ≤ 05:30; never exceed 06:00. Forward drift makes earlier safer.
- **Token expiry:** `CLAUDE_CODE_OAUTH_TOKEN` expires eventually → job fails → GitHub email surfaces it.
- **ToS:** 1 heartbeat/day is negligible volume; `CLAUDE_CODE_OAUTH_TOKEN` is Anthropic's supported CI path → low risk.

---

## 8. Success metrics

- ✅ Workflow runs green daily (~05:07 GMT+7; forward drift tolerated).
- ✅ **Settings → Usage shows the 5h window anchored ~05:00–05:30** (the real metric).
- ✅ 3 distinct sessions reachable across 09:30–12:00 and 13:15–17:45.
- ✅ Failure email received when the job fails.

## 9. Next steps & dependencies

- Dependency: active Claude subscription quota + valid `claude setup-token` token.
- After step-6 verification succeeds: optionally hard-pin all daily marks with extra cron lines (only if drift becomes a problem).
- If an always-on box (VPS/Pi) becomes available: migrate to real cron there (more punctual than GitHub).

## 10. Unresolved questions

1. Does a GitHub Actions (CI-token) heartbeat actually move the interactive usage window on this account? — confirm at step 6.
2. Keep `07 22` (05:07, max margin) or `30 22` (05:30, fatter morning S1)? — user preference; both yield 3 sessions.
3. Want push notification (Telegram/Slack) instead of email-only on failure? — currently email default.
