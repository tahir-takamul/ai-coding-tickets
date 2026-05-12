# Revert log — SonarQube rollout

> Updated as work lands. Each entry: what was created/changed, where it lives, how to undo.
> Apply logs (full `terraform plan`/`apply` output, helm output, kubectl events) are kept under `task/logs/` per step.

## Conventions

- Branch in `eng-infra` repo: `MIT-3467-sonarqube` (off `main`).
- Branch in `mithril` repo (this): `MIT-3467`.
- Both feature branches; revert = `git revert <sha>` or `git push origin :MIT-3467-sonarqube`.
- Terraform applies are **scoped with `-target=...`** to only the SonarQube resources — silmarils-cleanup drift in the same plan is intentionally NOT applied.
- All Azure Key Vault writes are reversible (KV soft-delete retains 90 days by default).

## State backups

Before any `terraform apply`, the live TF state is dumped to `task/logs/<step>-state-pre.json`. To restore:

```
cd /Users/mohd.tahir/DevWorkspace/eng-infra/<module>
terraform state push <task/logs/...-state-pre.json>
```

## Resources created — scoped TF apply against `eng-infra/cicd/terraform/azure/`

Apply command (planned):

```
terraform apply \
  -target=azurerm_user_assigned_identity.sonarqube \
  -target=azurerm_federated_identity_credential.sonarqube \
  -target=azurerm_key_vault_access_policy.sonarqube \
  -target=azuread_application.sonarqube_saml \
  -target=azuread_service_principal.sonarqube_saml \
  -target=azuread_service_principal_token_signing_certificate.sonarqube_saml \
  -target=azuread_app_role_assignment.sonarqube_saml_devops_admin
```

Created resources (filled in after apply):

| Resource | Address | ID / value | Revert command |
|---|---|---|---|
| Managed Identity | `azurerm_user_assigned_identity.sonarqube` | TBD (name: `mithril-mi-sonarqube`) | `terraform destroy -target=azurerm_user_assigned_identity.sonarqube` |
| Federated cred | `azurerm_federated_identity_credential.sonarqube` | TBD | `terraform destroy -target=azurerm_federated_identity_credential.sonarqube` |
| KV access policy | `azurerm_key_vault_access_policy.sonarqube` | TBD (against `mithril-shared-kv`, lower sub) | `terraform destroy -target=azurerm_key_vault_access_policy.sonarqube` |
| Entra app | `azuread_application.sonarqube_saml` | TBD (display: `SonarQube SSO`) | `terraform destroy -target=azuread_application.sonarqube_saml` |
| SP | `azuread_service_principal.sonarqube_saml` | TBD | `terraform destroy -target=azuread_service_principal.sonarqube_saml` |
| Signing cert | `azuread_service_principal_token_signing_certificate.sonarqube_saml` | TBD (thumbprint: TBD) | `terraform destroy -target=azuread_service_principal_token_signing_certificate.sonarqube_saml` |
| App role assignment | `azuread_app_role_assignment.sonarqube_saml_devops_admin` | TBD | `terraform destroy -target=azuread_app_role_assignment.sonarqube_saml_devops_admin` |

## Cloudflare scoped apply (T09) — `eng-infra/cloudflare/`

Append to `cicd_ingress_rules` adds three resources:

- `cloudflare_zero_trust_tunnel_cloudflared_config.cicd` (in-place: re-renders ingress JSON)
- `cloudflare_record.cicd_tunnel_ingress["sonarqube.takamul.cc"]` (DNS CNAME)
- `cloudflare_zero_trust_access_application.cicd["sonarqube.takamul.cc"]` (CF Access app)
- `cloudflare_zero_trust_access_policy.cicd["sonarqube.takamul.cc"]` (CF Access policy)

Apply scope (planned):

```
terraform apply \
  -target='cloudflare_zero_trust_tunnel_cloudflared_config.cicd' \
  -target='cloudflare_record.cicd_tunnel_ingress["sonarqube.takamul.cc"]' \
  -target='cloudflare_zero_trust_access_application.cicd["sonarqube.takamul.cc"]' \
  -target='cloudflare_zero_trust_access_policy.cicd["sonarqube.takamul.cc"]'
```

Revert: `terraform destroy` with the same `-target` flags after restoring the previous `terraform.tfvars` (drop the sonarqube entry from `cicd_ingress_rules`).

## Azure Key Vault writes (T03/T10) — `mithril-shared-kv` (lower sub)

Six secrets, all under tag `service=sonarqube,owner=cicd`:

| Secret name | Current value source | Notes |
|---|---|---|
| `cicd-sonarqube-admin-password` | python `secrets`-generated 28-char with special chars | T10: had to regenerate (initial openssl-base64 lacked special chars; Sonar password policy rejected) |
| `cicd-sonarqube-current-admin-password` | same as admin-password | T10: synced after manual rotation via `/api/users/change_password` so future helm-upgrade hooks succeed |
| `cicd-sonarqube-db-password` | `openssl rand -base64 24` | not actually used by chart (embedded mode — no separate Postgres pod) but harmless |
| `cicd-sonarqube-monitoring-passcode` | `openssl rand -base64 24` | |
| `cicd-sonarqube-saml-cert` | extracted from federation metadata XML at `https://login.microsoftonline.com/{tenant}/federationmetadata/2007-06/federationmetadata.xml?appid={appId}` (public Verify cert, sans BEGIN/END headers); SHA1 thumbprint `61:D8:7C:49:80:67:66:0A:E4:A6:40:1B:1C:A0:16:2C:25:FF:BD:11` matches Entra keyCredentials | T03 didn't need Rohit's TF state — public IdP metadata gave us the same cert |
| `cicd-sonarqube-token` | global analysis token `sqa_b0843b0f4c98e4df3d776d45bfdb1d9ed06e161a` (generated by user via Sonar UI on 2026-05-07; 1y expiry) | T10: replaces the placeholder pre-existing entry (dated 2026-03-12 from earlier work) |
| `cicd-github-packages-token` | Classic GitHub PAT, owner `tahir-takamul`, scopes `read:packages,repo,workflow`. Created in KV on 2026-05-10 (T14 / FINDING §F19c). | Needed by backend `./gradlew test` to pull `com.takamul:vault-crypto-*` from `maven.pkg.github.com/takamulai/vault-crypto-provider`. **Rotate after T14 close-out** — was pasted into the agent transcript in clear text. |

Revert: `az keyvault secret delete --vault-name mithril-shared-kv --subscription 7047239e-8819-4015-9344-5a7481a9439b --name <name>` per secret. Soft-delete retention 90d; `--purge` if needed (purge requires KV `Purge` permission — not on standard access policies).

To revoke the GH Packages PAT specifically: GitHub UI → Settings → Developer settings → Personal access tokens (classic) → revoke. Then re-mint a fresh one and `az keyvault secret set --vault-name mithril-shared-kv --name cicd-github-packages-token --value '<new-pat>'`. Re-run `./deploy.sh cloud playbooks/01-jenkins.yml` to refresh the K8s Secret. Jenkins controller picks up the new env on next pod restart; JCasC reload picks up the new credential value automatically.

## In-cluster K8s objects (T08) — namespace `cicd`

Created by `06-sonarqube.yml` (chart 2026.2.1, `community.enabled: true`):

- Helm release `sonarqube`
  - StatefulSet `sonarqube-sonarqube` (1 replica, Running)
  - Service `sonarqube-sonarqube` (ClusterIP `172.16.128.128:9000`)
  - PVC `sonarqube-sonarqube` (10Gi, Bound, managed-csi-premium) — embedded data store
- Secret `sonarqube-credentials` (managed by ansible) — keys: `password`, `currentPassword`, `db-password`, `monitoring-passcode`, `saml-certificate`, `secret.properties`
- ServiceAccount `sonarqube` (annotated `azure.workload.identity/client-id=4a6f4080-...`)

**No separate Postgres**: chart's `community.enabled: true` mode uses embedded persistent storage in place of a Postgres subchart, even when `postgresql.enabled: true` is also set. This is a chart 2025+ behavior change vs. the original PLAN/FINDING §F05 expectation.

In-Sonar configuration created by T10/T11 (not part of helm release; lives in Sonar's data dir on the PVC):

- **Admin password rotated** from `admin` to a 28-char password with special chars (in KV `cicd-sonarqube-admin-password`).
- **Sonar group `0309fec6-57e6-4446-bf4f-af084823d03a`** (the Takamul DevOps Admin Entra group ObjectId) created with global perms `admin, gateadmin, profileadmin, provisioning, scan`. Auto-populated by SAML group claim on login.
- **Project `smoke-test`** created via API for T11 verification. May be deleted after T11 if desired.
- **Global Analysis Token** `sqa_b0843b0f4c98e4df3d776d45bfdb1d9ed06e161a` generated (1y expiry) and stored in KV `cicd-sonarqube-token`.

Revert:

```
helm uninstall sonarqube -n cicd
kubectl -n cicd delete secret sonarqube-credentials
kubectl -n cicd delete pvc sonarqube-sonarqube
```

(PVC does **not** auto-delete on `helm uninstall` — must be deleted explicitly. Deleting the PVC erases Sonar history including any scans, projects, users, tokens — only the admin user record can be recreated; everything else needs to be re-set up in T10/T11.)

## ACR image (T07) — `mithrilacr.azurecr.io/cicd-agent`

| Tag | Digest | Notes |
|---|---|---|
| `sonar-14c6508` | `sha256:5c01ce38d46bd08b9a38b0f7fd1acf6d73f268e1107cc95e767636da21b93391` | Built/pushed 2026-04-27 with sonar-scanner-cli 6.2.1.4610 + openjdk17-jre-headless. Used by `:latest` post-T11. ~558 MB. |
| `sonar-jdk25-cabec7c` | `sha256:b89b393b6d260b1de41018991eaf58da68b9e928e53f6b9706561cbf8495376d` | Built/pushed 2026-05-08 from `eng-infra` HEAD `cabec7c`. openjdk25 only (`25.0.3_p9-r1`) replaces openjdk17-jre-headless. **Superseded** by the JDK 17+25 image below — kept in registry for rollback reference. FINDING §F19b. |
| `sonar-jdk25` (sliding) | (currently points to the same digest as `sonar-jdk25-cabec7c`) | Kept for rollback; sonar-scan.Jenkinsfile no longer references it (re-pinned to `:sonar-jdk17-25-bef7e41`). Safe to leave or delete. |
| `sonar-jdk17-25-27e44eb` | `sha256:c94a88be1dc09e310d8be57d99609fad3a6a77b451fc838d16020ade1e57367d` | Built/pushed 2026-05-10 from `eng-infra` HEAD `27e44eb`. **First image carrying both JDKs** (openjdk25 + openjdk17). Needed because backend's `keycloak-hybrid-pq-provider` pins `keycloakJavaVersion=17` (Keycloak 26 SPI compat); Gradle per-module toolchain selects by language version. Superseded by `bef7e41` below (which adds Node). Kept for rollback. |
| `sonar-jdk17-25-bef7e41` | `sha256:720efbd9a7dcd9ae58ba7cc1d9a2c9eb4eea750b569d8d7262588cb9d9970c61` | Built/pushed 2026-05-11 from `eng-infra` HEAD `bef7e41`. Adds nodejs (24.15.0) + npm (11.12.1) on top of openjdk25 + openjdk17 + sonar-scanner-cli 6.2.1. **Current production sonar pipeline image** — pinned by `mithril:MIT-3467` commit `581616eef` in `iac/eng-infra/cicd/pipelines/modules/sonar-scan.Jenkinsfile`. Image grew ~80 MB from the previous tag (nodejs + npm). FINDING §F19c. |
| `sonar-jdk17-25` (sliding) | (currently same digest as `sonar-jdk17-25-bef7e41`) | Kept up-to-date with each `:sonar-jdk17-25-<sha>` build but **not referenced by the Jenkinsfile** (which uses the SHA-pinned tag). Sliding tag exists for ad-hoc `docker run` debugging on workstations. FINDING §F19c (SHA-pin rationale). |
| `latest` | `sha256:5c01ce38d46bd08b9a38b0f7fd1acf6d73f268e1107cc95e767636da21b93391` (post-T11 retag, 2026-05-07) | **Unchanged by all the JDK / Node work** — five orchestrator pipelines (`deploy*`, `demo-orchestrator`, `qa-orchestrator`, `staging-orchestrator`) consume `:latest` and don't need any of it; we kept their runtime stable. Was `sha256:2954061b247b3a627f097daa34be497402530d80b170321a9f8b42961c00e8f3` (2026-03-12) pre-T11. |

T11 retag command (executed 2026-05-07):

```
az acr import --name mithrilacr --subscription 6fdc2633-addd-4eb2-98cf-2ff8425f0fe9 \
  --source mithrilacr.azurecr.io/cicd-agent@sha256:5c01ce38d46bd08b9a38b0f7fd1acf6d73f268e1107cc95e767636da21b93391 \
  --image cicd-agent:latest --force
```

**To revert `:latest` to the pre-T11 image** (the one without sonar-scanner):

```
az acr import --name mithrilacr --subscription 6fdc2633-addd-4eb2-98cf-2ff8425f0fe9 \
  --source mithrilacr.azurecr.io/cicd-agent@sha256:2954061b247b3a627f097daa34be497402530d80b170321a9f8b42961c00e8f3 \
  --image cicd-agent:latest --force
```

JDK 25 image build (executed 2026-05-08, FINDING §F19b):

```
cd /Users/mohd.tahir/DevWorkspace/eng-infra/cicd/docker/cicd-agent
SHA=$(git rev-parse --short HEAD)   # cabec7c at build time
docker build --platform linux/amd64 \
             -t mithrilacr.azurecr.io/cicd-agent:sonar-jdk25-${SHA} \
             -t mithrilacr.azurecr.io/cicd-agent:sonar-jdk25 .
az acr login --name mithrilacr
docker push mithrilacr.azurecr.io/cicd-agent:sonar-jdk25-${SHA}
docker push mithrilacr.azurecr.io/cicd-agent:sonar-jdk25
# DO NOT also tag :latest — sonar-scan.Jenkinsfile pins :sonar-jdk25 explicitly.
```

The `--platform linux/amd64` flag is required when building on Apple Silicon (the Dockerfile pulls an x86_64-only `oc` binary; AKS nodes are amd64 anyway). Build verify confirmed `javac 25.0.3` + `SonarScanner CLI 6.2.1.4610 / Java 25.0.3 Alpine (64-bit)`.

JDK 17+25 image build (executed 2026-05-10, FINDING §F19c) — adds JDK 17 alongside JDK 25:

```
cd /Users/mohd.tahir/DevWorkspace/eng-infra
az acr build --registry mithrilacr --platform linux/amd64 \
  --image cicd-agent:sonar-jdk17-25-27e44eb \
  --image cicd-agent:sonar-jdk17-25 \
  cicd/docker/cicd-agent/
```

JDK 17+25+Node image build (executed 2026-05-11, FINDING §F19c) — adds nodejs + npm:

```
cd /Users/mohd.tahir/DevWorkspace/eng-infra
az acr build --registry mithrilacr --platform linux/amd64 \
  --image cicd-agent:sonar-jdk17-25-bef7e41 \
  --image cicd-agent:sonar-jdk17-25 \
  cicd/docker/cicd-agent/
```

Both used `az acr build` (remote build) rather than local `docker build` because local Docker Desktop wasn't running. Either approach works.

**To revert the JDK 25 image swap** (back to the JRE-17 image for sonar pipelines):

```
# Option A: re-pin sonar-scan.Jenkinsfile back to mithrilacr.azurecr.io/cicd-agent:latest
#           (a one-line revert in iac/eng-infra/cicd/pipelines/modules/sonar-scan.Jenkinsfile),
#           which sends sonar back to the same JRE-17 image other pipelines run.
# Option B: rebuild :sonar-jdk25 from a Dockerfile commit before openjdk17→openjdk25.
```

Existing pipelines (`deploy.Jenkinsfile`, `deploy-perf.Jenkinsfile`, `demo-orchestrator.Jenkinsfile`, `qa-orchestrator.Jenkinsfile`, `staging-orchestrator.Jenkinsfile`) all reference `cicd-agent:latest` and **stay on the JRE-17 image** through the JDK 25 work — they are not retagged. Image is additive (~75–85 MB delta from pre-T07 baseline — adds OpenJDK + sonar-scanner CLI; nothing removed). The `:sonar-jdk25` image adds another ~+150 MB on top (full JDK vs. headless JRE) but is consumed only by the sonar pipeline.

Revert per-tag (don't usually need to): `az acr repository delete --name mithrilacr --image cicd-agent:<tag>`.

## Jenkins JCasC + Helm changes (T14) — `eng-infra:MIT-3467-sonarqube`

T14 added a GH Packages credential to Jenkins and (separately) captured a manual `kubectl patch` admin entry as code so helm SSA stops conflicting. All changes committed in `eng-infra:MIT-3467-sonarqube` (commits `de1849a` + `1348e26`); applied to the live cluster via `./deploy.sh cloud playbooks/01-jenkins.yml -- -e jenkins_helm_extra_args=--force`.

| What | Where | Revert |
|---|---|---|
| `github_packages_token` key in `cicd_secret_names` | `cicd/ansible/inventory/group_vars/all.yml` | `git revert de1849a` in `eng-infra` |
| `github_packages_token` field in `cicd-credentials` K8s Secret | `cicd/ansible/playbooks/01-jenkins.yml` (creates the Secret) | revert + re-run `./deploy.sh cloud playbooks/01-jenkins.yml` |
| `GITHUB_PACKAGES_TOKEN` env var on Jenkins controller container | `cicd/k8s/jenkins/values.yaml` controller.containerEnv | revert + helm upgrade |
| JCasC `usernamePassword` credential `github-packages` (username `tahir-takamul`) | `cicd/k8s/jenkins/values.yaml` JCasC.credentials block | revert + helm upgrade (Jenkins JCasC reloads automatically after the new ConfigMap renders) |
| `mohamed.sabith@takamul.ai` added to admin role in helm values | `cicd/k8s/jenkins/values.yaml` JCasC.role-based-auth (commit `1348e26`) | **DO NOT revert** without checking with whoever granted Mohamed admin originally. This entry codifies pre-existing runtime state (was added via manual `kubectl patch`); reverting would remove their admin in the next helm upgrade. If admin is to be revoked, do it intentionally not as a side-effect. |
| `--force` flag passed to helm upgrade (one-time, to break the SSA field-manager conflict on the `jenkins-jenkins-config-role-based-auth` ConfigMap) | runtime arg to `helm upgrade`, not a file change | N/A — no persistent effect; future runs don't need `--force` because helm now owns `f:data` for that ConfigMap |

To roll back all of T14's Jenkins-side changes at once:
```
cd /Users/mohd.tahir/DevWorkspace/eng-infra
git revert de1849a 1348e26
git push origin MIT-3467-sonarqube
cd cicd/ansible
./deploy.sh cloud playbooks/01-jenkins.yml
```

The Jenkins pipeline-side counterpart (the `withCredentials` block in `sonar-scan.Jenkinsfile`) lives in the `mithril` repo (commit `5a72c5f64`) and would also need to be reverted to fully back out the `github-packages` integration; otherwise the next backend scan would fail with "ERROR: Could not find credentials entry with ID 'github-packages'".

## T16 — GHA workflow + composite action (mithril:MIT-3467)

| What | Where | Revert |
|---|---|---|
| `.github/workflows/sonar-scan-pr.yml` | this repo (mithril caller) | `git rm .github/workflows/sonar-scan-pr.yml` — removes the mithril-side trigger. Reusable workflow + composite action stay available for any other repo's caller. |
| `.github/workflows/sonar-scan-reusable.yml` | this repo (shared reusable) | `git rm .github/workflows/sonar-scan-reusable.yml` — also removes the entry point any other repo would use. Coordinate with consuming repos first. |
| `.github/actions/sonar-pr-report/action.yml` | this repo | `git rm -r .github/actions/sonar-pr-report/` — composite action; lives nowhere else. |
| `task/REUSE.md` | this repo | onboarding doc; safe to keep even if reverting the workflows. |
| Five GHA repo secrets | github.com → Repo Settings → Secrets and variables → Actions | `gh secret delete <name>` per secret: `CF_ACCESS_CLIENT_ID`, `CF_ACCESS_CLIENT_SECRET`, `JENKINS_API_USER`, `JENKINS_API_TOKEN`, `SONARQUBE_READ_TOKEN`. CF Access and Jenkins API tokens may still be used by `trigger-jenkins-deploy.yaml` — verify before delete. |
| Jenkins API token (`gha-sonar-pr`) | Jenkins UI → User → Configure → API Tokens | Click **Revoke** on the token row in the Jenkins UI. |
| `SONARQUBE_READ_TOKEN` Sonar user token | Sonar UI → My Account → Security | Click **Revoke** on the token row. |

### T16 — Cloudflare Access policy (eng-infra:MIT-3467-sonarqube)

| What | Where | Revert |
|---|---|---|
| `cloudflare_zero_trust_access_policy.cicd_service_token_sonar` — authorizes `github-actions-jenkins-webhook` service token on `sonarqube.takamul.cc` | `eng-infra/cloudflare/main.tf` (commit `520c4de`) | `git revert 520c4de` in `eng-infra`, then `terraform apply -target=cloudflare_zero_trust_access_policy.cicd_service_token_sonar` in `eng-infra/cloudflare/`. Or destroy directly: `terraform destroy -target=cloudflare_zero_trust_access_policy.cicd_service_token_sonar`. |

After reverting just this policy, the existing CF Access app for `sonarqube.takamul.cc` continues to enforce the user-identity policy (SAML SSO) — Sonar UI access for humans is unchanged. Only the GHA workflow loses its API access.

## T15+T16 sticky issues to fix on merge to main

| File | Change |
|---|---|
| `iac/eng-infra/cicd/pipelines/modules/sonar-pr.Jenkinsfile` | `defaultValue: 'MIT-3467'` → `'main'` on `PIPELINE_BRANCH` |
| `iac/eng-infra/cicd/pipelines/modules/sonar-scan.Jenkinsfile` | same |
| `eng-infra/cicd/k8s/jenkins/values.yaml` JCasC `mithril-folder-sonar-{pr,scan}-job` | `stringParam('PIPELINE_BRANCH', 'MIT-3467', …)` → `'main'` for both |

After flipping the defaults, run `./deploy.sh cloud playbooks/01-jenkins.yml` to refresh JCasC.

## What is intentionally NOT applied in this rollout

The CICD TF plan includes ~6 destroy operations and 1 in-place AKS change (`upgrade_settings` block drift) that are **unrelated** silmarils-cleanup drift from earlier merges. Per user instruction: those are scoped out via `-target` flags and remain pending for whoever owns that cleanup.

| Resource | Status | Owner |
|---|---|---|
| `azurerm_private_dns_a_record.jenkins` | drift, NOT touched | platform team |
| `azurerm_private_dns_zone.internal` | drift, NOT touched | platform team |
| `azurerm_private_dns_zone_virtual_network_link.cicd` | drift, NOT touched | platform team |
| `azurerm_private_dns_zone_virtual_network_link.silmarils` | drift, NOT touched | platform team |
| `azurerm_virtual_network_peering.cicd_to_silmarils` | drift, NOT touched | platform team |
| `azurerm_virtual_network_peering.silmarils_to_cicd` | drift, NOT touched | platform team |
| `azurerm_kubernetes_cluster.cicd` (`upgrade_settings` block) | drift, NOT touched | platform team |
