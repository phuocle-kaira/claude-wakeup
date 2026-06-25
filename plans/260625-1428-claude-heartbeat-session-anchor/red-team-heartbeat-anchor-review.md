# Red-Team Review — Heartbeat Session Anchor (hard mode)

Adversarial pass over the plan. Brutal, concise. Verdict per finding: BLOCKING / FIX / WATCH / NON-ISSUE.

## F1 — Mechanism premise unproven (WATCH, gated)
Attack: the whole plan assumes a CI-token `claude -p "hi"` moves the *same* rolling 5h window as interactive usage. If subscription windows are tracked per-surface or CI usage is metered separately, the heartbeat anchors nothing.
Verdict: Already the plan's #1 risk and Phase 2 gates on Settings → Usage. Not blocking — but this is make-or-break. Do not trust green runs until Usage confirms once.

## F2 — 60-day auto-disable of scheduled workflows (FIX — new finding)
Attack: GitHub **automatically disables scheduled workflows after ~60 days of repository inactivity** (no commits). This repo's only purpose is the heartbeat → it will receive no other commits → the schedule will silently stop after ~60 days. Symptom: heartbeat just stops firing, no failure email (disable ≠ failure).
Mitigation options (low → high effort):
- Be aware: if Usage shows the morning anchor stopped, re-arm by pushing any commit or clicking "Enable workflow" in the Actions tab. Set a ~50-day calendar reminder.
- Or add a keep-alive: a step (or tiny second schedule) that commits a timestamp file with `permissions: contents: write`. More moving parts — only if manual re-arming proves annoying.
Verdict: Real gap, not covered by the email alert. Documented in plan + Phase 2. Default = awareness + reminder (YAGNI); keep-alive optional.

## F3 — DST / timezone drift (NON-ISSUE, verified)
Attack: if UTC↔local offset changes seasonally, `22:07 UTC` drifts off 05:07 local.
Check: Vietnam (Asia/Ho_Chi_Minh, GMT+7) has **no DST**; UTC has no DST. `22:07 UTC` = `05:07 GMT+7` **year-round, stable**. No drift.
Verdict: Non-issue here. (Would matter only for a DST locale.)

## F4 — Token leakage (NON-ISSUE)
Attack: token exposed in logs.
Check: token only via `secrets.CLAUDE_CODE_OAUTH_TOKEN` env; no `set -x`, no echo; `claude -p "hi"` does not print env; GitHub masks secrets in logs. User mints + sets it; Claude never handles it (hard rule).
Verdict: Non-issue.

## F5 — Quota cannibalization (NON-ISSUE)
Attack: burning S1 at 05:07 wastes quota the user needs.
Check: S1 (05:07–10:07) overlaps work only 09:30–10:07; its pre-work quota is unused anyway (user asleep). The "hi" call itself is negligible. Net: no meaningful loss — this is the intended trade.
Verdict: Non-issue; it's the design.

## F6 — Single heartbeat only fixes S1 (WATCH)
Attack: S2/S3 re-anchor on the user's own messages after 10:07 / 15:07; a long gap straddling a boundary shifts them.
Check: simulation shows anchors 05:07–06:30 all yield 3 sessions; a gap merely shifts S2/S3 later but still 3, as long as no S1 reset lands in lunch. Only the >07:00 anchor (S1 reset in lunch) drops to 2 — and that's prevented by the 05:07 target.
Verdict: Acceptable; bounded by the documented anchor ceiling.

## F7 — Cost / runner minutes (NON-ISSUE)
~1 min/day on ubuntu-latest. Well within free tier. Non-issue.

## Summary
- BLOCKING: none.
- FIX: F2 (60-day disable) — documented + mitigation added to plan/Phase 2.
- WATCH: F1 (verify in Phase 2), F6 (anchor ceiling holds it).
- NON-ISSUE: F3, F4, F5, F7.

Design survives. Proceed, with F1 verified and F2 understood before relying on it long-term.
