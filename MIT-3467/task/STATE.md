# State — SonarQube Community Build rollout (MIT-3467)

> **Read this first** when resuming with fresh context. Lists what is done,
> what is in flight, what's blocked, and exactly where to pick up.
>
> Companion files in this dir:
> - `PLAN.md` — overall rollout plan, scope, decisions.
> - `FINDING.md` — gotchas / constraints discovered during planning.
> - `T01.md`–`T16.md` — per-task specs (read for details when working a task).
> - `T-Rohit.md` — the one-time Terraform apply Rohit must run for T02.
> - `REVERT.md` — what's been changed and how to undo each piece.
> - `logs/` — apply logs, TF state backups.

---

## TL;DR — where we are (as of 2026-05-11, ALL TASKS DONE 🎉)

- ✅ T01–T13 all applied and verified live. SonarQube fully operational; legacy single-project config + GHA→Jenkins trigger deleted (commit `70b3eea5d`).
- ✅ Smoke scan passed (T11): `smoke-test` project analyzed; ncloc=4 visible in Sonar UI.
- ✅ **T14 DONE** (2026-05-11). Option B (6 per-branch project keys: `mithril-{backend,portal,wallet}-{main,staging}`). All three components scan green via `mithril/modular-jobs/sonar-scan`. Three tests skipped with documented reasons (FINDING §F19c).
- ✅ **T15 DONE** (2026-05-11). `sonar-pr.Jenkinsfile` wrapper + JCasC `mithril/modular-jobs/sonar-pr` + `generic-webhook-trigger` token `mithril-sonar-pr-token`. Webhook URL `https://jenkins.takamul.cc/generic-webhook-trigger/invoke?token=mithril-sonar-pr-token` (CF Access required).
- ✅ **T16 DONE** (2026-05-11). Layered into three pieces: caller workflow `.github/workflows/sonar-scan-pr.yml` (mithril-specific paths + triggers), reusable workflow `.github/workflows/sonar-scan-reusable.yml` (`workflow_call` interface for any takamulai repo to consume), composite action `.github/actions/sonar-pr-report/` (the actual trigger + poll + Sonar-read + Check-post logic). Triggers on PR open/sync into `[main, staging]` touching `backend|portal|wallet`. Per changed component: triggers Jenkins via CF-Access-tunneled webhook, polls, reads Sonar Web API for new_* + overall metrics, posts a GitHub Check + idempotent PR summary comment with 4-column table (Metric · New Code · Project (overall) · QG Threshold) + inline review comments scoped to `sinceLeakPeriod=true` issues. Validated end-to-end via real PR #1384 (rollout PR). New-repo onboarding documented in `task/REUSE.md`. All five GHA secrets in place (`CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`, `JENKINS_API_USER`, `JENKINS_API_TOKEN`, `SONARQUBE_READ_TOKEN`). Race condition between concurrent PR scans on the same base branch is accepted (FINDING §F28).

---

## Repos & branches

| Repo | Local path | Branch | Purpose | Pushed? |
|---|---|---|---|---|
| `takamulai/mithril` | `/Users/mohd.tahir/DevWorkspace/sonar-azure/mithril` | `MIT-3467` | T12, T13 file deletes; this `task/` directory | task/ untracked → commit before context clear |
| `takamulai/eng-infra` | `/Users/mohd.tahir/DevWorkspace/eng-infra` | `MIT-3467-sonarqube` | All TF / Helm / Ansible / Dockerfile changes | ✅ pushed — uncommitted: cloud.yml MI fill-in |

**Important note about scope split**: per `PLAN.md`, paths the plan calls `[SIBLING] cicd/...` map to `eng-infra/cicd/...` today, NOT the `iac/eng-infra/cicd/...` path that the plan/tasks reference. The eng-infra refactor (commit `046bebf73` on `mithril:main`, Apr 8 2026) moved that subtree out of `mithril` into the dedicated `takamulai/eng-infra` repo. Stale checkouts of `mithril` exist that still have the pre-refactor layout (e.g. `azure_jenkins/mithril/iac/eng-infra/...`); ignore those.

---

## Decisions made (deviations from PLAN/T-files worth knowing)

| Topic | Plan said | Chose | Why |
|---|---|---|---|
| Helm chart version | `10.8.0` | `2026.2.1` | Chart 10.x retired from repo; current is CalVer (FINDING §F23). |
| Schema for admin password | `account.adminPasswordSecretName` | `setAdminPassword.passwordSecretName` | Chart ≥2025 deprecated `account.*`. Secret keys: `password`, `currentPassword` (was `admin-password`, `current-admin-password`). |
| Image tag | `community` (floating) | unchanged — `community.enabled: true` + `image.tag: community` | User: "don't change sonar version". Chart pins community build via `community.buildNumber: 26.3.0.120487`; `image.tag: community` overrides if needed. |
| `entra_tenant_id` variable | new var, defaulted | not added | Existing pattern uses `data.azuread_client_config.current.tenant_id`. Mirrored. |
| Cloudflare tunnel name | `eng-infra-azure` | `eng-infra-cicd` | Plan referenced wrong tunnel (the `eng-infra-azure` tunnel is OKD's; the CICD AKS tunnel is `eng-infra-cicd`). |
| TF apply scope | full apply | **scoped `-target=...`** | The cicd & cloudflare TF state has unrelated drift from earlier merges (silmarils-dev cleanup); user instruction: don't touch existing infra in this rollout. |
| Sonar project keys | A: `mithril-{backend,portal,wallet}` (3 projects, main only) | **B: `mithril-{backend,portal,wallet}-{main,staging}` (6 projects)** | User chose during T14 verification (commit `8c3ab2105`). Jenkinsfile passes `-Dsonar.projectKey=…-${branch}` so per-module `.properties` defaults (target `-main`) are overridden at scan time. |
| `cicd-agent` JVM (T07 vs T14) | JRE 17 only ("no system JDK") | **Both openjdk25 AND openjdk17** | Backend's 16 modules use JDK 25; `keycloak-hybrid-pq-provider` pins `keycloakJavaVersion=17` (Keycloak 26 compat). Gradle per-module toolchain picks the right one. FINDING §F19b. |
| `cicd-agent:latest` retag for new image | retag `:latest` → JDK image | **Keep `:latest` on JRE-17 (sha 5c01ce38)**; sonar pipeline pins `:sonar-jdk17-25-<sha>` explicitly | 5 non-sonar orchestrators consume `:latest` incl. production deploys; JDK swap unnecessary for them and carries small regression risk. Sonar gets explicit immutable pin instead. |
| `cicd-agent` extras for T14 | sonar-scanner + JDK | **+ Node.js 24 + npm 11** | Wallet's sonar stage runs `npm install` so Sonar's TS analyzer resolves types from `node_modules`. FINDING §F19c. |
| `sonar-scan.Jenkinsfile` image tag | sliding tag | **immutable `:sonar-jdk17-25-<eng-infra sha>`** | K8s `imagePullPolicy=IfNotPresent` keeps nodes on cached layers for sliding tags; SHA pin forces fresh pulls and makes each pipeline run traceable to an eng-infra commit. |
| `org.sonarqube` Gradle plugin | T14.md template said `5.1.0.4882` | **`6.2.0.5505`** | 5.x calls `DslObject.getConvention()`, removed in Gradle 9. Backend uses gradle-wrapper 9.3.0 → must use 6.x line. |
| Backend GitHub Packages auth | not in PLAN/T14 | **Classic PAT in KV → JCasC `github-packages` credential → withCredentials** | Backend resolves `com.takamul:vault-crypto-*` from `maven.pkg.github.com`. Org policy blocks fine-grained PATs for org packages; used a classic PAT (`read:packages,repo,workflow`) owned by `tahir-takamul`. FINDING §F19c. |

---

## Resources actually created in Azure / ACR / KV (verified 2026-05-06)

### Azure (cicd subscription `6fdc2633-addd-4eb2-98cf-2ff8425f0fe9`)

| Resource | Address | Value |
|---|---|---|
| Managed Identity | `mithril-mi-sonarqube` (RG `mithril-rg-cicd`) | clientId `4a6f4080-7d5f-4211-a2ea-4967379df187`, principalId `443af500-2eaa-4509-93ca-1f80750528df` |
| Federated cred | `sonarqube-federated-cred` on the MI | subject `system:serviceaccount:cicd:sonarqube` |
| KV access policy | on `mithril-shared-kv` (lower sub) | objectId `443af500...` (the MI), perms `Get,List` |

### Azure Key Vault `mithril-shared-kv` (lower sub `7047239e-8819-4015-9344-5a7481a9439b`)

5 of 6 secrets seeded:

| Secret name | Status |
|---|---|
| `cicd-sonarqube-admin-password` | ✅ random 24-char base64, set 2026-04-27 |
| `cicd-sonarqube-current-admin-password` | ✅ literal `admin` (chart rotates to target on first sync — F10) |
| `cicd-sonarqube-db-password` | ✅ random 24-char base64 |
| `cicd-sonarqube-monitoring-passcode` | ✅ random 24-char base64 |
| `cicd-sonarqube-token` | ⚠️ pre-existing (from earlier work, dated 2026-03-12) — T10 overwrites with real Jenkins user token after first SAML login |
| `cicd-sonarqube-saml-cert` | ⛔ **NOT YET SEEDED** — needs T02 cert PEM from Rohit |

### ACR `mithrilacr.azurecr.io`

- `cicd-agent:sonar-14c6508` — built + pushed 2026-04-27. JRE-17 + sonar-scanner CLI 6.2.1.4610 build. Tag name reflects the pre-fix commit hash because the build was retried with the fix in the working tree before that fix was committed; the post-fix commit on the branch is `935e5c4`.
- `cicd-agent:latest` — retagged to `sha256:5c01ce38...` (= the `sonar-14c6508` digest) at T11 (2026-05-07). Consumed by 5 orchestrator pipelines; **stays on JRE-17 through the JDK 25 work**.
- `cicd-agent:sonar-jdk25-cabec7c` + sliding `cicd-agent:sonar-jdk25` — **built + pushed 2026-05-08**, digest `sha256:b89b393b6d260b1de41018991eaf58da68b9e928e53f6b9706561cbf8495376d` (~676 MB, +118 MB vs. the JRE-17 image). openjdk25 `25.0.3_p9-r1` replaces openjdk17-jre-headless. Build verified `javac 25.0.3` + scanner CLI 6.2.1 on Java 25. Consumed only by `sonar-scan.Jenkinsfile` (explicit pin). FINDING §F19b.

### Kubernetes (CICD AKS, namespace `cicd`)

- Nothing yet. T08 deploys via Helm.

### Cloudflare

- Nothing yet. T09 applies the new `cicd_ingress_rules` entry.

---

## Permissions state (`mohd.tahir@takamul.ai`, oid `3cebe077-64e5-49e1-95b9-19c58842d030`)

| Resource | Permission | Status |
|---|---|---|
| Entra ID directory roles | (any) | ❌ none — cannot create Entra apps. Reason T02 is blocked. |
| KV `mithril-shared-kv` | `Get, List, Set, Delete` on secrets | ✅ granted 2026-04-27 by self (Rohit set up the KV originally; mohd had owner-equivalent on RG, was able to self-grant access policy) |
| AKS `mithril-aks-cicd` | kubectl context | ✅ active context. Cluster RBAC not explicitly verified — assume working since Jenkins playbook was deployed from same workstation previously. |

| User | Notable roles |
|---|---|
| `rohit.verma@takamul.ai` | Application Administrator (Entra) — only person who can register apps without Global Admin help |
| `mahmoud.zein@takamul.ai`, `water.guo@TAKAMULDIGITAL.onmicrosoft.com`, `adm-mahmoud@TAKAMULDIGITAL.onmicrosoft.com` | Global Administrator (Entra) |

If long-term hygiene matters: ask one of the Global Admins to grant `mohd.tahir` the `Application Administrator` role so future TF runs (cert rotation, redirect URI changes) don't require Rohit. Forward block is in `T-Rohit.md` § "Path 2".

---

## Task graph & status

```
T01 ✅  done — MI mithril-mi-sonarqube + federated cred + KV policy live
T02 ✅  done (via T-Rohit) — Entra app SonarQube SSO appId 6fcb272d-629e-4050-8cee-776bc4a4484b
T-Rohit ✅ done — Rohit applied the 4 azuread_* resources from his machine on 2026-05-07
T03 ✅  done — 6/6 sonarqube KV secrets present
T04 ✅  done — group_vars populated; sonarqube_* values committed
T05 ✅  done
T06 ✅  done
T07 ✅  done — image pushed to ACR; :latest retagged to the sonar-scanner-bearing digest in T11
T08 ✅  done — helm release sonarqube live, status UP, version 26.4.0.121862
T09 ✅  done — CF tunnel + Access app/policy live for sonarqube.takamul.cc
T10 ✅  done — admin password rotated, SAML SSO works (after applicationId + claims fixes),
              Sonar group 0309fec6-... mapped to admin perms, jenkins-scanner global
              analysis token generated and pushed to KV, jenkins-0 restarted
T11 ✅  done — smoke pod scan succeeded; smoke-test project visible with measures
T12 ✅  done — root sonar-project.properties deleted (commit 70b3eea5d)
T13 ✅  done — .github/workflows/trigger-sonar-analysis.yaml deleted (same commit)
T14 ✅  done (2026-05-11) — Option B (6 per-branch projects). All three
              components (backend, portal, wallet) on `main` scan green.
              Sonar UI: `mithril-backend-main`, `mithril-portal-main`,
              `mithril-wallet-main` populated. Three tests skipped via
              `-x` / dropped step with documented reasons (F19c).
              Jenkinsfile last-pin: `:sonar-jdk17-25-bef7e41` (mithril
              commit `581616eef`).
T15 ✅  done (2026-05-11) — sonar-pr.Jenkinsfile wrapper (mithril
              commit `4a099f8ae`) + JCasC mithril/modular-jobs/sonar-pr
              + generic-webhook-trigger token `mithril-sonar-pr-token`
              (eng-infra commit `cc9096d`). Smoke-tested from in-cluster:
              wrapper #1 ran 2m 41s, dispatched portal sonar-scan ran
              green, wrapper inherited SUCCESS via wait+propagate.
T16 ✅  done (2026-05-11) — three-layer workflow:
              - sonar-scan-pr.yml (mithril caller; triggers + paths)
              - sonar-scan-reusable.yml (workflow_call interface for
                any takamulai repo)
              - .github/actions/sonar-pr-report/action.yml (logic)
              Polish pass added: pr_branch from BASE branch (so PRs
              into main publish to mithril-x-main); New Code metric
              keys; 4-column table with QG thresholds per row;
              sinceLeakPeriod=true on inline issues; parameterized
              project_key_prefix + jenkins_webhook_token for reuse.
              Validated end-to-end via PR #1384 (the rollout PR
              itself — real pull_request → main context).
              CF Access service token authorized on sonarqube.takamul.cc
              (eng-infra cloudflare terraform commit 520c4de).
              Five GHA secrets in place. Race condition between
              concurrent PR scans on same base accepted, see F28.
              Onboarding for new repos: task/REUSE.md.
```

---

## Critical "do not" list

- ❌ **Do not run `terraform apply` without `-target=...` flags** in the cicd or cloudflare modules. Both have unrelated drift from earlier merges that would be destructive (silmarils VNet peering, jenkins private DNS, etc.). User instruction: this rollout doesn't touch other infra.
- ❌ Do not modify anything under `iac/cicd-onprem/` in the `mithril` repo (different Jenkins, out of scope — FINDING §F18).
- ❌ Do not push the cert PEM into git. It lives only in TF state (encrypted backend) and KV.
- ❌ Do not bump Jenkins's `cicd-agent` image tag in `eng-infra/cicd/k8s/jenkins/values.yaml` until T11 (otherwise existing pipelines try to scan against a server that doesn't exist).
- ❌ **Do not retag `cicd-agent:latest` to the JDK 25 image** without explicit coordination — 5 orchestrator pipelines including production deploys consume `:latest` and the JDK swap is unnecessary for them. Sonar pipeline gets an explicit pin to `:sonar-jdk25` instead. FINDING §F19b.

---

## Exact pickup steps for fresh context

All tasks T01–T16 done. SonarQube rollout is fully operational on `main` AND on PR scans.

**What's live**:
- SonarQube server: `sonarqube-sonarqube.cicd.svc:9000` (in-cluster), `sonarqube.takamul.cc` (via CF tunnel + Access; service token authorized for GHA reads).
- Three projects analyzed: `mithril-backend-main`, `mithril-portal-main`, `mithril-wallet-main`. Per-branch sub-projects (`-staging` etc.) auto-create on first scan.
- Jenkins folder `mithril/modular-jobs/`:
  - `sonar-scan` (T14): runs the actual scan for one component. Pinned to `cicd-agent:sonar-jdk17-25-bef7e41`.
  - `sonar-pr` (T15): wrapper job, accepts webhook from GHA, dispatches sonar-scan with wait+propagate.
- GitHub Actions workflow `.github/workflows/sonar-scan.yml` (T16): fires on PR open/sync for paths `backend|portal|wallet/**`. Fanned matrix per changed component → composite action `.github/actions/sonar-pr-report` → trigger Jenkins → poll → fetch Sonar metrics → post Check + summary comment + inline review on diff lines.

**Known limitations / tech debt** (FINDING §F19c, §F26, §F27):
- **Tests skipped in `sonar-scan.Jenkinsfile` backend stage**: `:app:test` (Testcontainers needs Docker), `:keycloak-hybrid-pq-provider:test` (one test asserts K8s SA token file absent). Remove the `-x` flags once underlying issues are fixed by backend team.
- **Wallet jest suite entirely skipped**: 18/25 jest suites fail with env-leak issues (transformIgnorePatterns missing `@noble/hashes`, RN native module mocks missing). Wallet team fix; restoration is a one-line Jenkinsfile change to add `npm run test:coverage` back.
- **Wallet `npm install --legacy-peer-deps`**: needed because wallet's package.json has peer-dep conflicts (`react-native-screens` wants RN ≥0.82 but project pins 0.80). npm 11 enforces strict by default. Wallet team fix; remove the flag once their deps are aligned.
- **Sonar Community Build per-branch limitation** (FINDING §F16): only `main` is stored per project. PR scans publish under `mithril-${component}-main`, so each PR scan overwrites the previous. "New code" semantics are approximated by intersecting Sonar issues with the PR diff.

**Useful runbook entries**:
- **Manual sonar-scan run**: trigger `mithril/modular-jobs/sonar-scan` in Jenkins UI with `COMPONENT=<x> BRANCH=main PIPELINE_BRANCH=MIT-3467` (or `main` once merged).
- **Manual sonar-pr smoke test from outside**: `curl -X POST -H "CF-Access-Client-Id: $ID" -H "CF-Access-Client-Secret: $SECRET" -H "Content-Type: application/json" -d '{"component":"portal","commit_sha":"<sha>","pr_number":"0","pr_branch":"main"}' https://jenkins.takamul.cc/generic-webhook-trigger/invoke?token=mithril-sonar-pr-token`.
- **Manual GHA workflow dispatch**: GitHub UI → Actions → SonarQube Scan → Run workflow → pick component + paste commit sha.
- **Re-trigger CI for a PR**: push a no-op commit; the workflow re-runs and patches comments in place.

**Outstanding before merging MIT-3467 → main**:
- After merge, flip `PIPELINE_BRANCH` defaults in `sonar-pr.Jenkinsfile`, `sonar-scan.Jenkinsfile`, and the JCasC seed (`mithril-folder-sonar-{pr,scan}-job` blocks) from `MIT-3467` to `main`. One-line change × 3 files.
- Untracked logs in `task/logs/` are still untracked — decide to commit or .gitignore.

**Future work — T17 candidate: adopt `sonarqube-community-branch-plugin`**:
- Third-party plugin (`mc1arke/sonarqube-community-branch-plugin`, LGPL-3.0, release `26.4.0` matches our Sonar version) that adds native branch + PR analysis to Community Build. Resolves FINDING §F28 properly (the race condition currently accepted) and gives us real "new code" semantics scoped per-PR vs target.
- Migration is well-scoped (~5 hours): helm-upgrade Sonar with the plugin's javaagent flags, swap per-branch project keys for `sonar.branch.name` / `sonar.pullrequest.*` properties, optionally retire the composite action's PR decoration if we adopt the plugin's native decoration.
- Risk: not SonarSource-supported; one-maintainer dependency. Acceptable since we've already committed to Community Build long-term per FINDING §F28 mitigation discussion.
- See `task/SONAR.md` §9 for the full migration outline.

**If a wallet/portal/backend re-scan is needed**: trigger `mithril/modular-jobs/sonar-scan` with the desired `COMPONENT`. Pod pulls `cicd-agent:sonar-jdk17-25-bef7e41` (no policy changes needed). Each run takes ~10–14 min for backend, ~3 min for portal, ~5 min for wallet.

**If the cicd-agent image needs to be rebuilt**:
1. Edit `eng-infra/cicd/docker/cicd-agent/Dockerfile`, commit + push to `MIT-3467-sonarqube`. Note the new short SHA.
2. `az acr build --registry mithrilacr --platform linux/amd64 --image cicd-agent:sonar-jdk17-25-<new-sha> --image cicd-agent:sonar-jdk17-25 cicd/docker/cicd-agent/`
3. Update `iac/eng-infra/cicd/pipelines/modules/sonar-scan.Jenkinsfile` line `image: mithrilacr.azurecr.io/cicd-agent:sonar-jdk17-25-<new-sha>`. Commit + push to `mithril:MIT-3467`.
4. Trigger a sonar-scan to validate.

**Doc-only follow-up**: `task/logs/T14-jenkins-jcasc-rebranch.log`, `T14-jenkins-jcasc-update.log`, `cicd-agent-jdk25-build.log`, `cicd-agent-jdk17-25-build.log`, `cicd-agent-nodejs-build.log`, `T14-jenkins-jcasc-gh-packages.log` are still untracked. Decide whether to commit them or leave untracked; they're useful for diagnosing future regressions.

---

## Tools / environment (already verified working)

- `az` CLI authenticated to tenant `af9e95b7-...` (takamul.ai), default sub `cicd`.
- `kubectl` context = `mithril-aks-cicd`.
- `helm` v4.1.1 with `sonarqube` repo added.
- `terraform` v1.14.4 with cicd + cloudflare modules `init`-ed.
- `docker` (Docker Desktop running) — needed for T07 rebuilds if any.
- `ansible-playbook` core 2.20.5 (installed via brew).
- `gh` 2.86.0.
- `jq`, `curl`, `unzip`, `openssl`.
