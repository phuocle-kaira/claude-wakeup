---
phase: 1
title: "Provision Workflow"
status: done
effort: "S"
---

# Phase 1: Provision Workflow

<!-- Updated: Validation Session 1 - anchor 05:07 -> 05:09; added keep-alive commit (F2); email-only alerts -->

## Overview

Create the heartbeat GitHub Actions workflow file, commit, and push to `main`. The workflow runs `claude -p "hi"` daily at 05:09 GMT+7 and self-commits a keep-alive timestamp to avoid the 60-day auto-disable. In-repo only — no secrets, no token in any file.

## Requirements

- Functional: `.github/workflows/heartbeat.yml` exists with the content below; `cron: 09 22 * * *` (05:09 GMT+7); includes keep-alive commit step; committed `Add daily Claude heartbeat workflow`; pushed to `main`; workflow visible in the Actions tab.
- Non-functional: OAuth token must NOT appear in any file or command (only `secrets.CLAUDE_CODE_OAUTH_TOKEN`); no GitHub App; the only repo write the workflow performs is the keep-alive timestamp.

## Architecture

Scheduled workflow, single job on `ubuntu-latest`. Steps: checkout → install `@anthropic-ai/claude-code` → run `claude -p "hi"` (token from secrets) → keep-alive commit. `permissions: contents: write` lets the job push the keep-alive timestamp using the default `GITHUB_TOKEN` (bot push does not re-trigger the workflow → no loop). `concurrency` prevents overlap; `timeout-minutes` prevents a silent hang. Alerting = GitHub's built-in failure email (no extra code/secret).

## Related Code Files

- Create: `.github/workflows/heartbeat.yml`
- Created by the workflow at runtime (not in this commit): `.github/last-heartbeat.txt`

## Implementation Steps

1. Pre-flight: confirm default branch `main` and `.github/workflows/heartbeat.yml` absent. (The working tree currently contains the new `plans/` artifacts — that is expected; step 3 uses a scoped `git add` of only the workflow file.)
2. Create `.github/workflows/heartbeat.yml`:

   ```yaml
   name: claude-heartbeat

   on:
     schedule:
       - cron: "09 22 * * *"   # 22:09 UTC = 05:09 GMT+7 (next day) — off-peak minute
     workflow_dispatch:

   concurrency:
     group: claude-heartbeat
     cancel-in-progress: false

   permissions:
     contents: write            # keep-alive commit to reset the 60-day scheduled-workflow timer

   jobs:
     heartbeat:
       runs-on: ubuntu-latest
       timeout-minutes: 10
       steps:
         - name: Checkout
           uses: actions/checkout@v4
         - name: Install Claude Code
           run: npm install -g @anthropic-ai/claude-code
         - name: Send heartbeat
           env:
             CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
           run: claude -p "hi"
         - name: Keep-alive commit
           run: |
             date -u +"%Y-%m-%dT%H:%M:%SZ" > .github/last-heartbeat.txt
             git config user.name  "github-actions[bot]"
             git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
             git add .github/last-heartbeat.txt
             git commit -m "Update heartbeat keep-alive timestamp" || echo "nothing to commit"
             git push
   ```

3. `git add .github/workflows/heartbeat.yml` (scoped — do NOT `git add -A`; plan artifacts must not ride along)
4. `git commit -m "Add daily Claude heartbeat workflow"`
5. `git push origin main`
6. Report the commit hash. Confirm the workflow appears under the Actions tab.

## Success Criteria

- [x] `.github/workflows/heartbeat.yml` present with exact content (`cron: 09 22 * * *`, keep-alive step, `permissions: contents: write`)
- [x] Commit `Add daily Claude heartbeat workflow` pushed to `main`; commit hash `e23daeb`
- [x] Workflow listed in the Actions tab (= valid YAML) — `claude-heartbeat`, id `301969653`, active
- [x] No token string anywhere in the repo; no unrelated file committed (scoped `git add`; `plans/` left untracked)

## Completion (2026-06-25)

Done. `heartbeat.yml` created verbatim from plan, YAML validated (`safe_load` + ruby), committed `e23daeb`, pushed to `main`, confirmed active in Actions tab (0 runs — awaiting secret). Phase 2 (token + secret + Usage verify) is user-owned and remains pending.

## Risk Assessment

- **`contents: write` blast radius:** the workflow can push to `main`. Acceptable for a solo single-purpose repo; the only write is the timestamp file.
- **Daily keep-alive junk commits:** accepted trade for avoiding the 60-day auto-disable (user chose this in validation).
- **Keep-alive may not reset the timer:** bot (`GITHUB_TOKEN`) commits *should* count as repo activity (standard keep-alive pattern), but GitHub behavior is not guaranteed. Fallback: if the morning anchor disappears, re-arm manually (push any commit / "Enable workflow"). Still worth a ~50-day awareness check.
- **YAML/cron typo → workflow not registered.** Mitigation: confirm it shows in the Actions tab, do not assume.
- **`actions/checkout@v4`:** official GitHub action; pinned to major `v4`. Optionally pin to a full SHA for stricter supply-chain hygiene.
- **Rollback:** `git rm` the workflow (and `.github/last-heartbeat.txt`) + push, or disable the workflow in the Actions tab.
