# FINDING — MIT-5236

> Numbered gotchas discovered while wiring the orchestrator to sync wallet
> versionName from `/VERSION` and push the sync commit to `main` / `staging`.
> Each entry follows: **What / Why it matters / How we handle it / Do NOT**.

---

## F01 — GitHub Apps cannot bypass `pull_request` or `required_status_checks` rule types

**What**: Even when a GitHub App is listed in a Ruleset's `bypass_actors` with
`actor_type: Integration`, `bypass_mode: always`, and the installation has the
full set of relevant permissions
(`administration: write`, `pull_requests: write`, `checks: write`,
`contents: write`), a direct `git push` from the App **is still rejected** by:

- `pull_request` rule → `remote: error: GH013 ... Changes must be made through a pull request.`
- `required_status_checks` rule → `remote: error: GH013 ... Required status check "X" is expected.`

Empirically verified during MIT-5236 by:
1. Disabling the Ruleset → push succeeds (so identity is correct, bypass *can*
   work for the other rules in the same ruleset).
2. Adding pull_requests + checks write permissions → no change.
3. Same Ruleset with `non_fast_forward`, `deletion`, `copilot_code_review`
   rules: bypass *does* work for those.

**Why it matters**: GitHub's docs
([Granting bypass permissions](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets#defining-the-bypass-list))
state that an App needs only `administration: write` to be a bypass actor.
That's necessary but not sufficient — the bypass appears to be silently no-op
for `pull_request` and `required_status_checks` rule types. So any architecture
relying on "App pushes the sync commit directly to a branch with these rules"
will fail at runtime even when the docs say it should work.

**How we handle it**: Treat App-token direct-push as **only viable** if the
target branch has none of `pull_request` / `required_status_checks` in its
Rulesets. For branches that need both (which is most production branches), the
sync has to land via a different mechanism — either:

- (a) a *normal* PR that includes the wallet changes alongside the `/VERSION`
  bump (process discipline), or
- (b) the action opens a PR for the sync (auto-merge still subject to the same
  rules, but at least the App can push the branch and open the PR).

**Do NOT**: Spend more time grinding permissions / actor IDs. The behavior is
consistent and reproducible. Investigate (a) or (b) directly.

---

## F02 — The default `GITHUB_TOKEN` cannot bypass branch protection at all

**What**: The `GITHUB_TOKEN` that GitHub Actions provides to every workflow run
**cannot be added to any branch-protection / Ruleset bypass list**, period.
Even if `github-actions[bot]` (`id 41898282`) appears to be in the
`bypass_pull_request_allowances.users` list of a classic protection, pushes
authenticated by `GITHUB_TOKEN` are denied.

**Why it matters**: The original PR #1438's action used `GITHUB_TOKEN`
implicitly (no extra auth wiring). The first orchestrator run on main hit
`GH006: Protected branch update failed for refs/heads/main`. People reach for
`GITHUB_TOKEN` because it's already available and feels "official" — but it's
specifically *not* allowed to defeat branch protection by design.

**How we handle it**: Mint an installation token from a dedicated GitHub App
via [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
and rewrite the git remote URL to use that token (`https://x-access-token:${TOKEN}@github.com/...`).
That's PR #1506's mechanism. The App approach also subject to F01.

**Do NOT**: Try to configure `GITHUB_TOKEN` permissions in `permissions:` block
to defeat branch protection. The `contents: write` / `administration: write`
toggles there control what the token can *read/write* on Contents/Admin APIs,
not whether the workflow's pushes can bypass protection.

---

## F03 — Classic branch protection and Rulesets enforce as a UNION

**What**: If a repo has both a classic branch protection rule on `main` AND a
Ruleset targeting `main`, **both** are enforced; an actor must satisfy or
bypass **both** independently. Listing an App in only the Ruleset's
`bypass_actors` is not enough if classic protection also covers that branch —
classic's enforcement still blocks.

**Why it matters**: During MIT-5236 we created Rulesets first (intending them
to *replace* classic), then deleted classic afterwards. Between those two
steps, the App's push was still blocked by *classic* even though it bypassed
the Ruleset.

**How we handle it**: When migrating from classic → Rulesets, do it as two
discrete steps:
1. Create the Rulesets with the desired bypass list and verify they enforce
   correctly (e.g., test via an unprivileged actor that the rules still fire).
2. Then delete the classic rules.

There's a brief window where the branch has both layers; that's fine for human
contributors (rules still apply) but the App's bypass won't be effective until
step 2.

**Do NOT**: Try to add the App to *classic* protection's `bypass_pull_request_allowances.apps`
as a workaround. Even if it accepts the API call, the App still won't bypass
required status checks because classic protection has no per-rule bypass
mechanism for status checks (only PR review).

---

## F04 — App permission grants need installation-level *acceptance*, not just App-level *editing*

**What**: When you edit a GitHub App's `permissions` on its settings page and
click *Save*, the org's installation does **not** automatically pick up the
new permissions. The installation continues to operate with the *old*
permission set until an org admin visits the installation page
(`https://github.com/organizations/<org>/settings/installations/<installation_id>`)
and clicks **Review and accept new permissions**.

**Why it matters**: We hit this twice during MIT-5236 — once when adding
`administration: write`, once when adding `pull_requests` + `checks: write`.
Both times the GitHub API at `/orgs/<org>/installations` continued to report
the *old* permission set for several minutes, even though the App settings UI
showed the new permissions. Token-mint at runtime succeeds either way (the
token reflects the *installation's* current grant), so the only way to detect
the gap is via `gh api /orgs/<org>/installations --jq '...permissions'`.

**How we handle it**: After any App permission edit, **explicitly check the
installation's permissions via API** before retesting downstream behavior:

```bash
gh api /orgs/takamulai/installations \
  --jq '.installations[] | select(.app_id == 3778510) | .permissions'
```

If the new permission is missing, the org admin still has to accept.

**Do NOT**: Assume "Save changes" on the App settings page is sufficient.
Email notifications about the pending acceptance can be missed.

---

## F05 — Re-run attempt suffix wasn't propagated to `unified_tag` output (regression introduced by composite extraction)

**What**: The pre-PR-#1438 orchestrator's tag step appended `-${run_attempt}`
to the tag on re-runs AND wrote the suffixed value back to `$GITHUB_OUTPUT`.
The composite-action extraction (commit `daaa0d861`) dropped that re-output.
After the extraction, `generate-tag.outputs.unified_tag` stayed unsuffixed on
re-runs while the actual pushed tag had the suffix. The per-platform build
workflows then used the unsuffixed value as their `ref:` input — meaning they
either checked out the *previous attempt's* tag, or (if attempt 1 had failed
before tagging) failed with "ref not found".

**Why it matters**: Latent silent-stale-build risk plus a hard "unrecoverable
re-run" failure mode if attempt 1 dies before tagging.

**How we handle it**: PR #1506 moved the suffix-append into the
`Generate unified tag` step so the suffix is included in `$GITHUB_OUTPUT`
before anything reads it. The `Create and push git tag` step now just
consumes the already-final value.

**Do NOT**: Touch the order of suffix-append vs `$GITHUB_OUTPUT` write in
`.github/actions/release-tag/action.yml`. The current order is load-bearing.

---

## F06 — `[skip ci]` is load-bearing once we move off `GITHUB_TOKEN`

**What**: GitHub Actions has anti-recursion behavior: pushes authenticated by
`GITHUB_TOKEN` do **not** trigger workflows (so a workflow's own commit can't
re-trigger itself). Pushes authenticated by a **GitHub App** installation
token **do** trigger workflows normally.

**Why it matters**: The composite action commits the wallet sync with
`[skip ci]` in the message. While we were using `GITHUB_TOKEN`, `[skip ci]`
was belt-and-suspenders — anti-recursion already protected us. Once we
switched to the release-bot App token (PR #1506), `[skip ci]` became the
*only* mechanism preventing the orchestrator from re-triggering itself on
every sync commit. Remove it and the orchestrator goes into an infinite loop
(each run produces a sync commit that triggers a new run).

**How we handle it**: Keep the `[skip ci]` literal in the commit message
template in `release-tag/action.yml`. It's not cosmetic.

**Do NOT**: "Clean up" the commit message to remove `[skip ci]`. Don't
"improve" it by moving to a `gitmessage` template that strips trailers — the
trailer must reach origin.

---

## F07 — `bumpVersion.js` also rewrites `versionName` (not just build numbers)

**What**: PR #1438's description (and our review notes) said
`bumpVersion.js` patches "per-build numbers (`versionCode` / `CFBundleVersion`)"
in the CI workspace. That's an undercount —
[`wallet/scripts/bumpVersion.js`](https://github.com/takamulai/mithril/blob/main/wallet/scripts/bumpVersion.js)
also rewrites:
- `wallet/package.json` → `.version`
- `wallet/android/app/build.gradle` → `versionName`
- `wallet/ios/DCMobileApp/Info.plist` → `CFBundleShortVersionString`

every CI run, reading the new value from `/VERSION`.

**Why it matters**: It means the *shipped binaries* always had the correct
marketing version even when committed source was stale — which is why the bug
was never visible to store users. It also means `bumpVersion.js` and the new
`sync-wallet-version.sh` overlap on these three fields. Today they're
idempotent (both read `/VERSION`, both produce the same value), but if anyone
later changes one of them they need to update both.

**How we handle it**: Don't fight it; just keep both scripts pointing at
`/VERSION` as the single source of truth. A future cleanup could have
`bumpVersion.js` skip the name fields (since `sync-wallet-version.sh` owns
them by then). Out of scope for MIT-5236.

**Do NOT**: Add a different "marketing version" source (e.g., a CHANGELOG
header, a Jira release name) without updating both scripts together.

---

## F08 — Ruleset `bypass_actors[].actor_id` for `Integration` is the App ID, not the installation ID

**What**: When configuring `bypass_actors` for a GitHub App with
`actor_type: "Integration"`, the `actor_id` field is the **App ID** (3778510),
**not** the installation ID (133962046). Trying to PUT the ruleset with the
installation ID fails with:

```
422 Validation Failed
"Actor  integration must be part of the ruleset source or owner organization"
"Invalid bypass actor: '{{actor_id: 133962046}, {actor_type: Integration}}'"
```

**Why it matters**: GitHub's API docs say the field is "the ID of the actor"
without disambiguating. Easy to guess wrong.

**How we handle it**: Always use the **App ID** (the integer visible at the
top of the App's GitHub settings page) for `actor_id` when `actor_type` is
`Integration`. The installation ID is for `/orgs/<org>/installations/<id>`-style
endpoints, not for Ruleset bypass.

**Do NOT**: Swap to installation ID when bypass appears to be silently
no-op — that's almost certainly F01 (rule-type limitation), not an ID mismatch.
The 422 above is the only failure mode of using the wrong ID, and it's loud.
