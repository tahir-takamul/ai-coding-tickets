# STATE — MIT-5236 wallet versionName sync via orchestrator

**Status**: partial. Sync works on the *quiet path* (when wallet ↔ `/VERSION`
are already in sync); the *active path* (orchestrator commits + pushes the
sync) is blocked by a GitHub Ruleset bypass limitation (see [`FINDING.md`](FINDING.md) F01).

Last updated: 2026-05-20.

---

## TL;DR for fresh context

1. PR #1438 wired the orchestrator to commit + push a wallet versionName
   sync to `main` / `staging` before tagging.
2. PR #1506 switched the push to use a dedicated GitHub App
   (`takamul-release-bot`) so it can bypass branch protection.
3. We verified empirically that the App **cannot** bypass `pull_request` or
   `required_status_checks` rule types (see FINDING F01), even with the full
   set of relevant permissions (`administration`, `pull_requests`, `checks`,
   `contents` all write). Bypass *does* work for other rule types.
4. Today the wallet versionName on `main` matches `/VERSION = 1.1.0`, so the
   orchestrator's sync step exits early with "already in sync; no commit
   needed" and the bypass code path is never exercised. Pipeline runs green.
5. The next `/VERSION` bump (or any wallet/versionName drift on a PR that
   doesn't include the sync) will re-trigger the failure unless we change
   the architecture.

---

## What is in place today

### Code (`takamulai/mithril`)

| Component | Where | Status |
|---|---|---|
| Sync script | `scripts/sync-wallet-version.sh` | live |
| Composite action | `.github/actions/release-tag/action.yml` | live, uses release-bot App token |
| Main orchestrator | `.github/workflows/release-orchestrator-main.yml` | passes `app-id` + `app-private-key` from secrets |
| Staging orchestrator | `.github/workflows/release-orchestrator-staging.yml` | same as main |
| Per-platform builds | `android-apk-build.yml`, 3× `ios-*-build.yml` | take `ref:` input, checkout the unified tag |

### GitHub App

| Detail | Value |
|---|---|
| Name | `takamul-release-bot` |
| App ID | `3778510` |
| Installation ID (on `takamulai` org) | `133962046` |
| Bot user ID | `286273348` |
| Installed on | `takamulai/mithril` (selected) |
| Permissions (post acceptance) | `administration: write`, `checks: write`, `contents: write`, `metadata: read`, `pull_requests: write` |
| Private key | stored in repo secret `RELEASE_BOT_APP_KEY` |
| App ID stored as | repo secret `RELEASE_BOT_APP_ID` |

### Rulesets on `takamulai/mithril`

| ID | Name | Target | Bypass actors |
|---|---|---|---|
| 16632034 | `release-bot-bypass-main` | `refs/heads/main` | `takamul-release-bot` (always) |
| 16632035 | `release-bot-bypass-staging` | `refs/heads/staging` | `takamul-release-bot` (always) |
| 10728108 | `Copilot review for default branch` | `~DEFAULT_BRANCH` (= main) | `takamul-release-bot` (always) — *added by MIT-5236* |
| 13122668 | `PR only + merge commits` | malformed pattern, doesn't apply | unchanged |
| 12477451 | `gitleaks-secret-scan` | malformed pattern, doesn't apply | unchanged |

Classic branch protection on `main` and `staging` was **deleted** as part of
this work; protection is now Ruleset-only.

---

## What is not in place

### The action's commit-and-push path is broken for `/VERSION` bumps

If a PR merges to `main` that bumps `/VERSION` without also updating the
three wallet name fields, the orchestrator's sync step will commit the diff
locally and try to push. The push will fail with `GH013` (see F01). Recovery
requires a follow-up PR with the wallet sync.

### Long-term architecture decision still pending

Three options on the table, none yet picked (user wants to decide later):

1. **"Fail loudly" in the action.** Drop the commit + push; if sync is needed,
   exit non-zero with a clear error message. Add a CI check on PRs that runs
   `sync-wallet-version.sh` against the head and fails if the result diffs
   from what's committed. Forces sync to happen in human-authored PRs.
2. **PR-based architecture.** Action creates a sync branch (non-protected ref,
   App can push fine), opens a PR, auto-merges via App. Subject to the same
   ruleset rules but at least merge-via-PR is the supported path.
3. **Skip for now.** Leave the failing-push behavior. Document that `/VERSION`
   bumps must include the wallet sync. Revisit when `/VERSION` next moves.

---

## Pickup steps for fresh context

1. Read [`FINDING.md`](FINDING.md) first — F01 is the load-bearing constraint
   shaping all design options.
2. Confirm current installation permissions:
   ```bash
   gh api /orgs/takamulai/installations \
     --jq '.installations[] | select(.app_id == 3778510) | .permissions'
   ```
   Should show all five (admin/checks/contents/metadata/pull_requests). If
   anything's missing, App permission edits weren't accepted on the
   installation (see F04).
3. Confirm Ruleset bypass actors:
   ```bash
   for id in 16632034 16632035 10728108; do
     gh api "/repos/takamulai/mithril/rulesets/$id" \
       --jq '{name, bypass_actors}'
   done
   ```
   Each should list `actor_id: 3778510, actor_type: Integration, bypass_mode: always`.
4. Confirm classic protection is still deleted:
   ```bash
   gh api /repos/takamulai/mithril/branches/main/protection 2>&1 | head -2
   gh api /repos/takamulai/mithril/branches/staging/protection 2>&1 | head -2
   ```
   Both should return 404 "Branch not protected".
5. Confirm wallet ↔ `/VERSION` parity on main:
   ```bash
   cat VERSION
   grep -m1 'versionName ' wallet/android/app/build.gradle
   grep -A1 'CFBundleShortVersionString' wallet/ios/DCMobileApp/Info.plist
   grep '"version"' wallet/package.json | head -1
   ```
   All four should agree. If not, see [`REVERT.md`](REVERT.md) §"Wallet
   versionName drift" to open a fix-up PR.

---

## Relevant PRs

| # | Title | Status |
|---|---|---|
| [#1438](https://github.com/takamulai/mithril/pull/1438) | `chore(release): sync wallet ios/android versionName in orchestrator` | merged |
| [#1506](https://github.com/takamulai/mithril/pull/1506) | `feat(release-tag): push sync commit using release-bot GitHub App token` | merged |
| [#1509](https://github.com/takamulai/mithril/pull/1509) | `test: revert wallet versionName to 0.4.0 to exercise App bypass` | merged (diagnostic; left main with stale 0.4.0) |
| [#1510](https://github.com/takamulai/mithril/pull/1510) | `chore(wallet): restore versionName to 1.1.0` | **open** — restores main after the diagnostic |
