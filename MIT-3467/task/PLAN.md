# Plan — SonarQube Community Build 24.12.0 on CICD AKS

> Master plan for this effort. Kept in sync with refinements as they land.
> Agents: read `FINDING.md` before starting any task.

## Goal

Stand up a **SonarQube Community Build 24.12.0** server on the existing **CICD AKS** cluster (the same cluster that already runs Jenkins, in namespace `cicd`), backed by a bundled PostgreSQL subchart, fronted by the existing Cloudflare tunnel `eng-infra-azure`, SSO'd through Azure Entra ID SAML the same way Jenkins is, and consumable by Jenkins pipelines via a `sonar-scanner` CLI baked into the existing `cicd-agent` image. Scope: **lower (non-prod) only**. CI = GitHub Actions, CD/tests = Jenkins.

This rollout delivers the **server stand-up only**. Per-module scan integration (`mithril-backend`, `mithril-portal`, `mithril-wallet`), GHA PR decoration, and quality-gate enforcement in CI are explicit follow-ups (see §Follow-ups), out of scope for this rollout.

## Scope split: this repo vs sibling infra repo

The infrastructure for this rollout lives in **two repositories**:

| Component | Repo | Path |
|---|---|---|
| Helm values, Ansible playbook, Terraform (MI + SAML), `cicd-agent` Dockerfile, group_vars | **azure_jenkins** sibling repo | `/Users/mohd.tahir/DevWorkspace/azure_jenkins/mithril/iac/eng-infra/` |
| Cloudflare tunnel + Access app/policy | platform-team repo (not on disk) | `cloudflare/` |
| Old root `sonar-project.properties` (delete), legacy GHA workflow (delete) | **this repo** (mithril) | `mithril/` |

For brevity, paths in tasks are written relative to the `eng-infra/` root and prefixed `[SIBLING]` when they live in `azure_jenkins`. Paths prefixed `[CF]` live in the cloudflare repo (out-of-band work — coordinate with platform/CF team). Paths with no prefix live in the current repo (`sonar-azure/mithril/`).

## Decisions (user-confirmed)

| Area | Choice |
|---|---|
| Cluster | CICD AKS (same cluster as Jenkins) |
| Namespace | `cicd` (same as Jenkins) |
| DB | Bundled `postgresql` subchart in-cluster on `managed-csi-premium` |
| Helm chart | `sonarqube/sonarqube` (SonarSource) |
| Image tag | `community` (Community Build, monthly rolling — no LTA) |
| SAML app | Terraform via `azuread` provider (new to this repo — Jenkins was manual) |
| PR decoration | **No** community-branch-plugin; main-branch analysis only |
| SSO | Azure Entra ID SAML, values shape mirrors Jenkins JCasC |
| Hostname | `sonarqube.takamul.cc` via existing Cloudflare tunnel `eng-infra-azure` |
| Workload identity | New MI `mithril-mi-sonarqube`, federated to SA `sonarqube:cicd`, KV `Get,List` perms |
| Scanner integration | `sonar-scanner` CLI baked into existing `cicd-agent` image |

## Architecture summary

```
Jenkins agent pod (cicd namespace, CICD AKS)
    │  sonar-scanner / ./gradlew sonar
    │  sonar.host.url=http://sonarqube-sonarqube.cicd.svc:9000   ← in-cluster Service DNS
    │  sonar.token=<from Jenkins credential, sourced from KV>
    ▼
SonarQube 24.12 + bundled Postgres
   (StatefulSet in cicd namespace, ClusterIP service)

Browser users
    │
    ▼
Cloudflare Tunnel `eng-infra-azure`  →  https://sonarqube.takamul.cc
    │  (Cloudflare Access policy gates the URL)
    ▼
Same SonarQube ClusterIP service
    │  → "Log in with Azure Entra ID" → SAML flow → logged in via Entra
```

- Jenkins ↔ SonarQube: **CLI only** (`sonar-scanner`, `./gradlew sonar`), over the **in-cluster Service DNS** `http://sonarqube-sonarqube.cicd.svc:9000`. No CF tunnel hop — same cluster, same namespace.
- Browser/human ↔ SonarQube: external hostname `sonarqube.takamul.cc` through Cloudflare Tunnel + Cloudflare Access; user identity then via Entra SAML.

## Critical anchor files (existing — `[SIBLING]` repo)

- `[SIBLING] cicd/ansible/playbooks/01-jenkins.yml` — playbook template to mirror for sonarqube
- `[SIBLING] cicd/ansible/playbooks/inc/fetch-and-store-secret.yaml` + `inc/credential-stores/azure_kv/get-secret.yaml` — Key Vault fetch helper (reuse)
- `[SIBLING] cicd/k8s/jenkins/values.yaml` (~lines 226-240) — SAML JCasC block to mirror in shape
- `[SIBLING] cicd/k8s/jenkins/values.yaml` (~lines 622-634) — storage + workload-identity SA pattern
- `[SIBLING] cicd/terraform/azure/identity.tf` (~lines 9-73) — MI + federated cred + KV policy template
- `[SIBLING] cicd/terraform/azure/providers.tf` — add `azuread` provider here
- `[SIBLING] cicd/ansible/inventory/group_vars/all.yml` (~lines 22-30) — `cicd_secret_names` dict
- `[SIBLING] cicd/ansible/inventory/group_vars/cloud.yml` (~lines 10-11) — `key_vault_name: mithril-shared-kv`
- `[CF] cloudflare/terraform.tfvars` (~lines 241-248) — `cicd_ingress_rules`
- `[CF] cloudflare/main.tf` (~lines 406-440) — access app + policy auto-pickup via `for_each`
- `[SIBLING] cicd/docker/cicd-agent/Dockerfile` — add `sonar-scanner` here

## New files to create

- `[SIBLING] cicd/terraform/azure/saml-sonarqube.tf` — `azuread_application` + `azuread_service_principal` + signing cert + group assignment
- `[SIBLING] cicd/ansible/playbooks/06-sonarqube.yml` — deploy playbook
- `[SIBLING] cicd/k8s/sonarqube/values.yaml` — Helm values

## Task index

| ID | Subject | Repo | Status | Blocked by |
|---|---|---|---|---|
| T01 | Terraform — workload identity for SonarQube (MI + federated cred + KV policy) | `[SIBLING]` | pending | — |
| T02 | Terraform — Entra SAML app via `azuread` provider | `[SIBLING]` | pending | — |
| T03 | Seed Azure Key Vault secrets (`cicd-sonarqube-*`) | out-of-band | pending | T02 (signing cert from TF output) |
| T04 | Ansible group_vars updates (`all.yml` + `cloud.yml`) | `[SIBLING]` | pending | T01, T02 |
| T05 | Helm values `cicd/k8s/sonarqube/values.yaml` | `[SIBLING]` | pending | — |
| T06 | Ansible playbook `06-sonarqube.yml` | `[SIBLING]` | pending | T05 |
| T07 | Add `sonar-scanner` CLI to `cicd-agent` Dockerfile + push to ACR | `[SIBLING]` | pending | — |
| T08 | Run `06-sonarqube.yml` against CICD AKS | `[SIBLING]` | pending | T01–T06 |
| T09 | Cloudflare tunnel + Access app/policy for `sonarqube.takamul.cc` | `[CF]` | pending | T08 |
| T10 | First-login hardening — rotate admin pwd, generate Jenkins user token, push to KV, re-run `01-jenkins.yml` | mixed | pending | T08, T09 |
| T11 | End-to-end verification (Sonar UP, SAML login, Jenkins smoke pipeline) | mixed | pending | T07, T10 |
| T12 | Delete root `sonar-project.properties` (this repo) | mithril | pending | T11 |
| T13 | Delete `.github/workflows/trigger-sonar-analysis.yaml` (this repo) | mithril | pending | T11 |
| T14 | Per-module integration: 3 Sonar projects + per-module `sonar-project.properties` + reusable `sonar-scan.Jenkinsfile` module | mixed | pending | T11 |
| T15 | `sonar-pr.Jenkinsfile` + JCasC webhook `mithril-sonar-pr-token` (Azure Jenkins) | `[SIBLING]` | **gated** | T14 + explicit user approval |
| T16 | GHA workflow `sonar-scan.yml` + composite action `.github/actions/sonar-pr-report` (this repo) | mithril | **gated** | T15 + explicit user approval |

## Rollout order (matches T01–T13)

1. T01–T02: Terraform — MI + federated cred + KV policy + Entra SAML app (`[SIBLING] cicd/terraform/azure/`)
2. T03: Seed KV secrets (admin pwd, db pwd, monitoring passcode, SAML IdP cert)
3. T04: Ansible `group_vars` additions
4. T05–T06: Helm `values.yaml` + new playbook `06-sonarqube.yml`
5. T07: Rebuild `cicd-agent` image with `sonar-scanner` → push to ACR
6. T08: Run `06-sonarqube.yml` against CICD AKS
7. T09: Cloudflare `terraform apply` adds tunnel rule + Access app/policy
8. T10: First-login hardening — rotate admin pwd; generate user token in UI; push token to KV as `cicd-sonarqube-token`; re-run `01-jenkins.yml` so Jenkins env refreshes
9. T11: End-to-end verification
10. T12–T13: Clean up legacy artifacts in this repo
11. T14: Per-module integration — three projects + reusable Jenkins scan module
12. **GATE: explicit user approval that T01–T14 are working as expected**
13. T15: `sonar-pr.Jenkinsfile` + JCasC webhook (Azure Jenkins)
14. **GATE: explicit user approval that T15 is working as expected**
15. T16: GHA workflow + composite action for PR-time feedback

## Critical file paths in this repo

### Delete (T12, T13)

- `sonar-project.properties` (repo root) — superseded; this rollout doesn't add a replacement (per-module configs are a follow-up)
- `.github/workflows/trigger-sonar-analysis.yaml` — points at legacy `dsocodescan.cbuae.gov`; not migrated as part of this rollout

### Strictly off-limits

- `iac/cicd-onprem/**` — different Jenkins deployment, out of scope (see FINDING.md §F18)
- All files under `mithril/iac/eng-infra/shared-okd/**` — that subtree corresponds to a different cluster topology (OKD) and is **not** the deployment target. The OKD anchor files in this tree are unused by this rollout.

## Open decisions (need user input before implementation starts)

1. **Backend Java version vs SonarQube image** (FINDING §F19) — Community Build 24.12.0 does not support Java 25 (added in 26.2+). Backend code is on Java 25. Per-module scan integration is **not in scope for this rollout**, so this decision is deferred — but must be answered before the follow-up "per-module scan integration" work begins. Options:
   - (A) Use the latest Community Build image (≥26.2.0-community). Strong recommendation (Community Build has no LTA — F03).
   - (B) Set `sonar.java.source=21` in `backend/sonar-project.properties`. Accepts partial analysis of Java 22-25 syntax.
   - (C) Defer Sonar rollout for backend until image is bumped.

2. **Cloudflare repo location & owner** (T09) — the `cloudflare/terraform.tfvars` and `cloudflare/main.tf` files referenced in this plan are not on the local machine. The CF Tunnel + Access change needs to be coordinated with whoever owns that repo. Confirm path and approval flow before T09.

3. **Entra group mapping beyond admin** (Follow-ups) — currently only the `Takamul DevOps Admin` group (`0309fec6-57e6-4446-bf4f-af084823d03a`) gets wired. If we want `developer` / `reader` tiers like Jenkins (mirrored from `[SIBLING] cicd/k8s/jenkins/values.yaml` role-based-auth), we need two more group object IDs and corresponding SonarQube "Groups" mapped via SAML. Add post-stand-up.

4. **Per-branch project keys** (T14) — Community Build can only retain one analysis per project. With two long-lived branches (`main`, `staging`), options:
   - (A) **Main-only** — three projects: `mithril-backend`, `mithril-portal`, `mithril-wallet`. Staging gets no Sonar dashboard; staging PRs see scanner output only via Jenkins logs (or via T16 GHA comments if those land). Default in T14.
   - (B) **Per-branch keys** — six projects: `mithril-{backend,portal,wallet}-{main,staging}`. Doubles dashboards + tokens + quality-gate config; staging history is preserved separately.
   - Decision deferred to start of T14. T14 is written for option A; switching to B is a name-string change in T14's `sonar-project.properties` files + Jenkinsfile param.

5. **Project-scoped tokens vs single user token** (T14, T16) — T10 generates one user token (`cicd-sonarqube-token`) used by Jenkins for all scans. This works because the user is a Sonar admin. Hardening: split into three project-analysis tokens (`cicd-sonarqube-token-{backend,portal,wallet}`) — narrower blast radius if one leaks, but adds a Jenkins re-run after T14 creates the projects. Decide before T14 step "generate tokens" runs.

## Out of scope (this rollout)

- HA/multi-replica SonarQube (not supported by Community Build).
- Native Sonar PR decoration (not supported by Community Build).
- `sonarqube-community-branch-plugin` (rejected — maintenance burden).
- PVC backup/restore automation.
- Token auto-rotation.
- `sdk` and `qa` module scans.
- Quality-gate enforcement that **blocks merges** in branch protection (T16 posts a check; opt-in to require the check is a separate GitHub repo-settings change).

## Follow-ups

- **Entra group mapping beyond admin** — see Open decision 3.
- **AKS node capacity** — `Standard_D2as_v4` = 8 GiB / 2 vCPU. Sonar recommends 4 GiB JVM heap; Jenkins is already on this pool. Watch memory pressure; may need a dedicated node pool if co-tenancy gets tight.
- **Token auto-rotation** — initial Jenkins token is manual (log in, generate, paste to KV). Follow-up playbook can POST to `/api/user_tokens/generate` via admin token and push to KV.
- **Backup story** — bundled postgres PVC is snapshot-only. Revisit if Sonar history ever becomes load-bearing (vs. re-scannable from source).
- **Monthly image bump** — Community Build has no LTA; treat `image.tag: community` as a regularly-bumped dependency.
- **Project-scoped tokens** — see Open decision 5; split single user token into three analysis tokens once T14 has created the projects.
- **`sdk` / `qa` module scans** — same pattern as T14 if/when the team wants them.
