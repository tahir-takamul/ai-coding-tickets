# Findings — gotchas & constraints for this work

> Read this file before starting any task. These are lessons captured from discovery so we don't relearn them.
> Add new findings as they emerge; never delete — mark as superseded if wrong.

## F01. Cluster + namespace = same as Jenkins

**What**: SonarQube deploys to the **CICD AKS** cluster, in the **`cicd` namespace** — the same cluster and namespace where Jenkins already runs.

**Why it matters**: Jenkins agents are pods in this same namespace; they reach SonarQube via the in-cluster Service DNS `http://sonarqube-sonarqube.cicd.svc:9000`. No Cloudflare tunnel hop, no peered-network DNS, no Routes. The previous design assumed a separate OKD cluster with internal/external dual-hostnames and OpenShift Routes; that is **not** the deployment topology.

**How we handle it**: K8s objects (Helm release, SA, Secrets, PVCs) all land in `cicd`. The `cicd_secret_names` dict in `[SIBLING] cicd/ansible/inventory/group_vars/all.yml` is shared across Jenkins + SonarQube; running the SonarQube playbook will re-fetch Jenkins keys (harmless overlap by design).

**Do NOT**: create a separate namespace. Don't deploy to `shared-okd` even though that subtree exists in the sibling repo — it corresponds to a different cluster topology.

## F02. SonarQube Community Build has no PR/branch analysis

**What**: Community Build (the free edition, renamed from "Community Edition" in Dec 2024) analyzes **main branch only**. Pull-request analysis, branch analysis, and PR decoration are paid-edition features. The `sonarqube-community-branch-plugin` is explicitly rejected for this rollout (maintenance burden).

**Why it matters**: Running `sonar-scanner -Dsonar.branch.name=feature/foo` will not create a separate branch dashboard; it will overwrite the main project's last analysis. Passing `-Dsonar.pullrequest.key=123` does nothing useful.

**How we handle it**:
- Stand-up scope is server only. Per-module scanning is a follow-up; PR decoration is also a follow-up.
- When PR-time feedback is later added, it will be driven from **GitHub Actions** by reading the Sonar Web API and posting GitHub Checks + PR review comments. Sonar itself only ever stores the latest analysis on the main project.

## F03. Community Build release model — monthly rolling, no LTA

**What**: Community Build ships **monthly** on CalVer (`YY.M.0.BuildNumber`). There is **no LTA/LTS** — LTA versions are exclusive to paid SonarQube Server.

**Why it matters**: Pinning to a specific tag and forgetting means missing bug fixes and rule updates. Upgrades arrive with no curated stable track.

**How we handle it**: Use the floating `community` tag in the Helm values for this initial stand-up. Schedule a monthly reminder to bump and re-run the playbook. If a build-pinned tag is preferred for change control, set `image.tag` to a specific build (e.g. `24.12.0.100206-community`) in `[SIBLING] cicd/k8s/sonarqube/values.yaml` and bump it explicitly each month.

## F04. Community Build feature gaps that affect this design

**No**: branch/PR analysis, taint-flow injection detection, SCA, SCIM auto-provisioning, audit logs, portfolios, security reports, PDF/regulatory reports, GitHub/GitLab native PR decoration, AI features, HA/clustering, IP allow-list, max-token-lifetime.

**Yes**: main-branch quality gate, bugs/code-smells/vulnerabilities tracking (non-taint), coverage import, standard rule packs, user/group permissions, webhooks, Web API, **SAML SSO** (Community Build does ship basic SAML — confirmed against the official feature-comparison table).

Do **not** propose designs that rely on any "No" feature without first flagging the Community-Build constraint.

## F05. Use the SonarSource Helm chart with bundled Postgres

**What**: The actively-maintained chart is `sonarqube` from `https://SonarSource.github.io/helm-chart-sonarqube`. Bitnami does not publish a SonarQube chart.

**Why it matters**: The chart ships a `postgresql` subchart (Bitnami Postgres bundled). For this rollout we **enable** the bundled Postgres (`postgresql.enabled: true`), against `managed-csi-premium`. This differs from the previous OKD-era plan that disabled the bundled DB and ran a separate Bitnami Postgres release; user-confirmed decision to keep Postgres bundled here.

**How we handle it**: Add the SonarSource Helm repo in the playbook. Use floating `image.tag: community`. Configure `postgresql.auth.existingSecret: sonarqube-credentials` so the user-password and admin-password come from the credentials secret created by the playbook from KV.

## F06. Jenkins ↔ Sonar: **CLI only**, in-cluster Service DNS

**What**: Inside the AKS cluster, Jenkins agents reach SonarQube via Kubernetes Service DNS at `http://sonarqube-sonarqube.cicd.svc:9000`. They use `sonar-scanner` / `./gradlew sonar`; they do **not** call the Sonar Web API for result retrieval.

**Why it matters**:
- The scanner CLI cannot inject Cloudflare Access headers (see F07), so it must not be pointed at the external `sonarqube.takamul.cc` URL.
- In-cluster Service DNS bypasses the tunnel entirely — fast, simple, no headers.
- Quality-gate enforcement on the CLI side: `-Dsonar.qualitygate.wait=true` exits non-zero on failure (works without the Jenkins Sonar plugin). `waitForQualityGate()` Groovy step requires the plugin and `withSonarQubeEnv()`.

**How we handle it**: pipelines in scope of this rollout do exactly one thing — call `sonar-scanner` against the in-cluster Service DNS using a token sourced from a Jenkins credential. The credential is fed from KV `cicd-sonarqube-token`, populated in T10.

## F07. `sonar-scanner` cannot traverse Cloudflare Zero Trust

**What**: `sonar-scanner`, `./gradlew sonar`, and `mvn sonar:sonar` speak plain HTTP to `sonar.host.url` and **do not support injecting custom headers** (e.g. `CF-Access-Client-Id` / `CF-Access-Client-Secret`). They cannot be pointed at `https://sonarqube.takamul.cc` from any caller — internal or external — because CF Access will reject requests without service-token headers.

**Why it matters**: This forces the architectural split — scanner traffic uses the internal Service DNS (F06); only browsers and Web API callers (which can inject headers) use the external Cloudflare URL.

**How we handle it**:
- All scanner invocations from Jenkins agents target `http://sonarqube-sonarqube.cicd.svc:9000`.
- If/when GHA PR decoration is added (follow-up): GHA uses `curl` (with header injection) against the external URL.

**Do NOT**:
- Add a sonar-scanner step to any GHA workflow that runs on hosted runners.
- Try to reverse-proxy the scanner's HTTP calls through a header-injecting sidecar without first confirming the approach.
- Expose SonarQube publicly to avoid the tunnel (security regression).

## F08. Workload identity follows the Jenkins pattern

**What**: SonarQube needs read access to the same Azure Key Vault (`mithril-shared-kv`) that Jenkins reads from, so the playbook can fetch admin password / DB password / SAML cert at deploy time. The pattern is workload-identity-on-AKS, mirroring `[SIBLING] cicd/terraform/azure/identity.tf` lines 9-73 (the Jenkins MI block).

**Why it matters**: SAS keys / SP secrets / kubeconfig-pinned creds are not used here. The pod surfaces an OIDC token, which Azure exchanges for an MI token, which has KV `Get,List` perms — no static credentials anywhere.

**How we handle it**:
- New MI `mithril-mi-sonarqube` (`azurerm_user_assigned_identity`).
- New federated credential with subject `system:serviceaccount:cicd:sonarqube`, issuer = AKS OIDC URL (reuse the existing data source in the Terraform module).
- New KV access policy on `mithril-shared-kv` via `provider = azurerm.lower`, secret_permissions `["Get","List"]`.
- The playbook patches SA `sonarqube` in `cicd` with annotation `azure.workload.identity/client-id={{ sonarqube_workload_identity_client_id }}` after Helm install; pod label `azure.workload.identity/use: "true"` is in the Helm values.

## F09. Entra SAML — Terraform `azuread` provider is new to this repo

**What**: Jenkins's Entra SSO was set up manually via the Azure portal. SonarQube's SAML app will be created via Terraform using the `hashicorp/azuread` provider — first use of that provider in this Terraform stack.

**Why it matters**:
- `[SIBLING] cicd/terraform/azure/providers.tf` must be updated to add the provider; the rest of the stack uses `azurerm`.
- A new variable `entra_tenant_id` (default `af9e95b7-e99f-4ec1-9b02-429ffdc27b4e` — the Jenkins tenant) is added.
- The signing certificate generated by `azuread_service_principal_token_signing_certificate` is the source of truth for the SAML cert that goes into KV; this is the chicken-and-egg-resolving artifact.

**How we handle it**:
- TF resources: `azuread_application.sonarqube_sso`, `azuread_service_principal.sonarqube_sso`, `azuread_service_principal_token_signing_certificate.sonarqube_sso`, group assignment to `Takamul DevOps Admin` (object id `0309fec6-57e6-4446-bf4f-af084823d03a`, pulled from Jenkins role-based-auth in `[SIBLING] cicd/k8s/jenkins/values.yaml:245`).
- TF outputs: `sonarqube_sso_application_id`, `sonarqube_sso_entity_id`, `sonarqube_sso_login_url`, `sonarqube_sso_idp_metadata_url`.
- The cert PEM body (sans BEGIN/END headers, as SonarQube expects) lands in KV as `cicd-sonarqube-saml-cert`.

## F10. Bootstrap admin password chicken-and-egg

**What**: SonarQube ships with default admin/admin. The chart can rotate to a configured password on first sync, but it needs to know the **current** password (i.e. `admin`) to authenticate the rotation.

**Why it matters**: KV must hold both `cicd-sonarqube-admin-password` (the target) and `cicd-sonarqube-current-admin-password` (literally the string `admin` at first boot). After first successful sync the chart updates "current" silently; the playbook should treat both as required-but-changeable.

**How we handle it**:
- T03 (KV seed) sets `cicd-sonarqube-current-admin-password` = `admin` literal on first boot.
- After T08 deploy, Sonar's admin password is the target value. Operators can rotate by updating `cicd-sonarqube-admin-password` and re-running the playbook; the chart needs `current-admin-password` to match the current value at the time of re-run.
- The Helm values reference `account.adminPasswordSecretName: sonarqube-credentials` with both keys in the same Secret.

**Do NOT**: hard-code passwords in values; both values come from KV via `inc/fetch-and-store-secret.yaml`.

## F11. KV secret naming convention — `cicd-sonarqube-*`

**What**: The existing convention seen in `cicd_secret_names` is `cicd-<service>-<key>`. SonarQube secrets follow:
- `cicd-sonarqube-admin-password` — target admin password (random)
- `cicd-sonarqube-current-admin-password` — literal `admin` on first boot
- `cicd-sonarqube-db-password` — bundled Postgres user/admin password (same value used for both per chart's `existingSecret` model)
- `cicd-sonarqube-monitoring-passcode` — chart's monitoring endpoint passcode
- `cicd-sonarqube-saml-cert` — x509 PEM body from `azuread_service_principal_token_signing_certificate`, sans headers
- `cicd-sonarqube-token` — Jenkins user token; placeholder until T10 fills it after first login

**Why it matters**: Add these to the `cicd_secret_names` dict so the playbook's secret-fetch loop picks them up. The dict is shared with Jenkins by design — no harm in extra keys.

## F12. Sonar SAML config requires the cert as a Secret (not inline)

**What**: SonarQube's `sonar.auth.saml.certificate.secured` value can exceed inline-property length limits. The Helm chart accepts a Secret (`sonarSecretProperties: sonarqube-credentials`) whose key `secret.properties` is appended to `sonar.properties` at boot. That's where the cert (and any other secret-bearing properties) lives.

**Why it matters**: The playbook constructs `secret.properties` content inside the K8s Secret (templated from KV), so the cert PEM body and the SAML application ID land together. Inlining the cert in `sonarProperties` doesn't work for typical x509 lengths.

**How we handle it**:
- The K8s Secret `sonarqube-credentials` has key `secret.properties` whose body looks like:
  ```
  sonar.auth.saml.certificate.secured=<PEM body, single line, no BEGIN/END>
  ```
- The Helm values key `sonarSecretProperties: sonarqube-credentials` is set; SonarQube concatenates that file at boot.
- The application ID can stay in plain `sonarProperties` since it's not secret.

## F13. AKS pod security — workload identity label + SA annotation are non-negotiable

**What**: For the workload identity exchange to work, the **pod** must carry label `azure.workload.identity/use: "true"`, and the **ServiceAccount** must carry annotation `azure.workload.identity/client-id=<MI client id>`. Without both, the pod gets `STS authentication failed` from Azure.

**Why it matters**: The chart's `serviceAccount.create: true` + `serviceAccount.name: sonarqube` is what Helm produces; the playbook patches the annotation post-install (matches the pattern used for Jenkins). Pod label is set declaratively in Helm values via `podLabels`.

**How we handle it**: see T05 (Helm values) and T06 (playbook). Verification: `kubectl -n cicd get sa sonarqube -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}'` must match the MI client id from Terraform output.

## F14. Cloudflare tunnel is the existing `eng-infra-azure` tunnel — same cnameTarget as Jenkins

**What**: SonarQube's external URL `sonarqube.takamul.cc` resolves through the existing Cloudflare tunnel `eng-infra-azure`. New rule appended to `cicd_ingress_rules` in `[CF] cloudflare/terraform.tfvars`; the Access app + policy auto-generates via the `for_each = local.cicd_access_apps` pattern in `[CF] cloudflare/main.tf:406-440`.

**Why it matters**:
- No new tunnel — saves a control-plane component.
- Access app picks up the same SSO policy used for Jenkins (Entra-tied), so user-facing SSO is "free" — but **Sonar still also enforces its own SAML** layer. Two layers (CF Access + Sonar SAML) is the same shape as Jenkins.

**How we handle it**: T09 adds the rule + applies the `[CF]` Terraform out-of-band. Coordinate with the platform/CF repo owner before pushing.

## F15. Hostname pattern — vanity subdomain `sonarqube.takamul.cc`

**What**: External hostname is the vanity-subdomain pattern `sonarqube.takamul.cc` (matches `jenkins.takamul.cc`). No `.apps.<cluster>` wildcard hostname is involved (that pattern is OKD-specific and not used in AKS).

**How we handle it**: 
- DNS for `sonarqube.takamul.cc` is auto-created by the Cloudflare TF module (CF-managed DNS).
- SonarQube's `sonar.core.serverBaseURL` = `https://sonarqube.takamul.cc` — used by SAML for asserting AssertionConsumerService URL and for redirects.

## F16. Community Build does not accept `-Dsonar.branch.name`

**What**: Passing `sonar.branch.name` to the scanner on Community Build is silently ignored for storage purposes — all analyses overwrite the main project.

**Why it matters**: When per-module scans are integrated later (follow-up), don't build pipeline logic that depends on finding a per-branch project in Sonar.

**How we handle it**: For this rollout's smoke test (T11), do not pass `sonar.branch.name`. For follow-ups: leave it out, or pass it harmlessly for forward-compat with a paid upgrade.

## F17. `sonar-scanner` CLI is added to the existing `cicd-agent` image

**What**: Rather than build a dedicated scanner image, append `sonar-scanner-cli-6.x` to `[SIBLING] cicd/docker/cicd-agent/Dockerfile`. The base image (`alpine/k8s:1.32.11`) already provides `unzip` and `curl`. Sonar Scanner CLI 6.x bundles its own JRE, so no JDK is needed for the scanner itself (Gradle/Maven projects bring their own toolchain).

**Why it matters**:
- One image, one tag bump. Keeps Jenkins pod templates stable (no new container in pod spec for scans).
- Avoids the previous-plan dependency on building two new ACR images (`sonar-scanner-jvm`, `sonar-scanner-node`).

**How we handle it**: T07 adds the install block (download zip, unzip to `/opt`, symlink to `/usr/local/bin`, smoke `sonar-scanner --version`). Bump the image tag in Jenkins agent pod template **after** the server is up (T11+).

## F18. `iac/cicd-onprem/` is OUT OF SCOPE

**What**: The `mithril/iac/cicd-onprem/` directory contains pipelines deployed to a different Jenkins instance (on-prem CBDCR SIT cluster). Modifying files there will affect a separate deployment that is not owned by this work stream. User explicitly confirmed: "no need to read anything in mithril/iac/cicd-onprem as it is unrelated".

**How we handle it**: Read-only reference is forbidden; writes are forbidden. New scanner integration goes only against the AKS Jenkins.

**Do NOT**: edit, import, or reference any file under `iac/cicd-onprem/`.

## F19. Java 25 across server and agent — both resolved 2026-05-08

Two distinct Java-25 problems, often conflated:

### F19a — SonarQube server analyzer (RESOLVED)

**What**: Community Build 24.12.0 predates Java 25; analyzer support landed in Community Build **26.2+**.

**Resolution**: T08 deployed `sonarqube 26.4.0.121862` (Helm chart `2026.2.1`, see F23). Server-side analyzer support for Java 25 is in place.

### F19b — cicd-agent JDK for `./gradlew test` (RESOLVED)

**What**: T07's `cicd-agent` Dockerfile installs only `openjdk17-jre-headless` — a JRE, not a JDK — to give sonar-scanner CLI a JVM. Backend's `./gradlew test` declared via `gradle.properties javaVersion=25` (Gradle toolchain) needs a JDK 25 with `javac`. T14's first backend scan (Jenkins job `mithril/modular-jobs/sonar-scan` build #5) failed:

```
Toolchain installation '/usr/lib/jvm/java-17-openjdk' does not provide
the required capabilities: [JAVA_COMPILER]
```

**Resolution (2026-05-08)**: Replace `openjdk17-jre-headless` with full `openjdk25` in `eng-infra/cicd/docker/cicd-agent/Dockerfile`. Single JDK serves both purposes — sonar-scanner CLI is plain Java bytecode and runs on any JVM ≥ 11; backend's `./gradlew test` gets the `javac` it needs. Reverses T07's "no system JDK" stance only because the input changed (real JDK consumer arrived); the spirit (don't install JDKs gratuitously) still holds.

**Rollout strategy — `:latest` is NOT retagged**: `cicd-agent:latest` is consumed by five non-sonar pipelines today (`deploy.Jenkinsfile`, `deploy-perf.Jenkinsfile`, `demo-orchestrator.Jenkinsfile`, `qa-orchestrator.Jenkinsfile`, `staging-orchestrator.Jenkinsfile`) — including production deploys. The JDK 25 swap is unnecessary for them and carries small but real regression risk (different `JAVA_HOME` path, +150 MB pull on first node hit). To keep blast radius limited to sonar:

1. Build + push the JDK 25 image with two tags: an immutable `cicd-agent:sonar-jdk25-<sha>` (for revert reference) and a sliding `cicd-agent:sonar-jdk25` (what the Jenkinsfile pins to).
2. **Do NOT** `az acr import` to `:latest`. The five orchestrator pipelines keep running on the JRE-17 image at `sha256:5c01ce38...` (T11 retag).
3. `sonar-scan.Jenkinsfile` pins the pod-template image explicitly to `mithrilacr.azurecr.io/cicd-agent:sonar-jdk25`. No other Jenkinsfile changes needed.

If/when the JDK 25 image proves stable across all sonar runs (1–2 weeks), promotion to `:latest` becomes optional — there's no functional reason `:latest` needs to be the JDK build, so leaving the split indefinitely is fine.

**Bloat**: ~+150 MB on the JDK 25 image (vs. JRE 17). One-time pull cost per AKS node thanks to layer caching. Only sonar pipelines pay it under this rollout strategy.

**How sonar-scan.Jenkinsfile uses it**: backend stage runs `./gradlew clean test jacocoTestReport sonar` directly inside the `cicd-agent` container — the org.sonarqube Gradle plugin uploads in-process, no separate `sonar-scanner` invocation needed for backend. Portal/wallet still use `sonar-scanner` CLI from the same container; the scanner is plain Java bytecode, runs fine on JDK 25.

**Do NOT**:
- Revert to a JRE-only image without first migrating backend off this container or wiring Gradle toolchain auto-download (`foojay-resolver-convention` + agent egress to `api.foojay.io`).
- Retag `:latest` to the JDK 25 image without coordinating with whoever owns the five orchestrator pipelines — `JAVA_HOME` path change is the most likely break.

### F19c — T14 environmental drift: skipped tests, Gradle plugin bump, GH Packages auth (RESOLVED)

Discovered during T14's run-to-green on 2026-05-10/11. Five things had to be done that weren't in the PLAN/T14 spec because the Jenkins agent pod is a different runtime environment than the test suites were written for. Recording here for future agents and the wallet/backend teams:

**1. Three tests skipped (env leaks)** — the test code makes assumptions that don't hold inside a K8s pod:

| Skip path | Symptom | Root cause | Surgical fix |
|---|---|---|---|
| `./gradlew test -x :app:test` | `IllegalStateException at DockerClientProviderStrategy.java:274` for 190/206 tests | Testcontainers needs a Docker daemon; AKS uses containerd, no Docker socket exposed in the pod | Add Testcontainers Cloud or a remote `DOCKER_HOST` envvar; OR tag these tests `@DisabledIfEnvironmentVariable(KUBERNETES_SERVICE_HOST)` |
| `./gradlew test -x :keycloak-hybrid-pq-provider:test` | 1/83 test fails: `VaultClientTest.k8sLogin_failedAuth_throwsException` AssertionError | Test asserts error message "Kubernetes service account token" — only fires when `K8S_TOKEN_PATH` (`/var/run/secrets/kubernetes.io/serviceaccount/token`) is absent. In a K8s pod, that file IS mounted, so the K8s login proceeds, hits the mock server, fails with "Vault K8s login failed" instead | Make `K8S_TOKEN_PATH` configurable (env var fallback) and point to a non-existent path in the test; OR use `MockedStatic<Files>` to mock `readK8sServiceAccountToken()` |
| Drop `npm run test:coverage` for wallet entirely | 18/25 jest suites fail | Mix of: (a) `transformIgnorePatterns` doesn't transform ESM from `@noble/hashes`, `@noble/curves`, `@noble/post-quantum`, `js-base64`; (b) native-module mocks missing for `react-native-encrypted-storage`, `@sbaiahmed1/react-native-biometrics`, `react-native-config`; (c) 3-4 stale assertions where source drifted | Three changes by wallet team: add the noble/js-base64 packages to `transformIgnorePatterns`; add `jest.mock(...)` setup for native RN modules; fix stale assertions in `Activity`, `FilterRangeSelection`, `NotificationDisplayService`, `PaymentErrorScreen` tests |

Net effect on Sonar coverage:
- Backend: 15 of 17 modules contribute coverage (`app/` and `keycloak-hybrid-pq-provider/` show 0%). Static analysis runs for all 17.
- Wallet: 0% coverage (jest entirely skipped). Static analysis runs.
- Portal: 0% coverage (no jest configured today — that's by design, not a skip).

**Why the pattern repeats**: the test suites were written for GitHub Actions runners (no K8s, has Docker for Testcontainers, isolated JS environment). Moving test execution from GHA → Jenkins-pod (PLAN's consolidation choice for T14) surfaces every env assumption. Each skip is one Jenkinsfile flag/line; un-skipping each requires source/test fixes in the respective team's domain. The split is deliberate: the rollout doesn't block on test-portability fixes the teams will do on their own schedule.

**2. `org.sonarqube` Gradle plugin bumped to 6.2.0.5505**:

T14.md's template specified `5.1.0.4882`. That version calls `org.gradle.api.internal.plugins.DslObject.getConvention()`, which was removed in Gradle 9. Backend uses `gradle-wrapper.properties` pinned to `9.3.0`, so the 5.x plugin fails immediately:

```
* What went wrong:
'org.gradle.api.plugins.Convention org.gradle.api.internal.plugins.DslObject.getConvention()'
```

Resolution: bump to `6.2.0.5505` in `backend/build.gradle` (commit `e13b8b713`). The 6.x line dropped the `Convention` API and supports Gradle 9. Pairs cleanly with the sonar-scanner CLI 6.2.x already in the cicd-agent image.

**3. Backend needs GitHub Packages auth in Jenkins**:

`backend/build.gradle` resolves `com.takamul:vault-crypto-spring-boot-starter` and friends from `https://maven.pkg.github.com/takamulai/vault-crypto-provider`. Without credentials, `./gradlew test` fails with `Username must not be null!` on the wallet-service module.

The legacy GHA pipeline ([deleted in T13] `.github/workflows/backend-unit-test.yaml`) used `GITHUB_ACTOR` + `secrets.PACKAGES_READ_TOKEN`. Jenkins had no equivalent — the only GitHub credential in JCasC was `github-ssh` (SSH key for git clone, not a PAT for packages).

Resolution (built end-to-end in T14, see commits below):
1. **Mint a classic PAT** with `read:packages` scope, owned by a takamulai org member. Fine-grained PATs don't work without an org-level toggle (403 "Resource not accessible by personal access token").
2. **Store in KV**: `cicd-github-packages-token` in `mithril-shared-kv`.
3. **Wire through Ansible/JCasC**:
   - `cicd/ansible/inventory/group_vars/all.yml`: add `github_packages_token` to `cicd_secret_names`.
   - `cicd/ansible/playbooks/01-jenkins.yml`: add `github_packages_token` key to the existing `cicd-credentials` K8s Secret.
   - `cicd/k8s/jenkins/values.yaml`: add `GITHUB_PACKAGES_TOKEN` env var (from secret); add JCasC `usernamePassword` credential `github-packages` (username hardcoded to the PAT owner, password from `${GITHUB_PACKAGES_TOKEN}`).
4. **Bind in `sonar-scan.Jenkinsfile`** backend stage: `withCredentials([usernamePassword(credentialsId: 'github-packages', usernameVariable: 'GITHUB_ACTOR', passwordVariable: 'GITHUB_TOKEN')])`.
5. **Apply**: `./deploy.sh cloud playbooks/01-jenkins.yml -- -e jenkins_helm_extra_args=--force`. The `--force` is needed because `kubectl-patch` previously owned the `jenkins-jenkins-config-role-based-auth` ConfigMap's `f:data` field (someone had manually added `mohamed.sabith@takamul.ai` to admin via patch); helm SSA refused to apply with a field-manager conflict until the same admin entry was added to values.yaml (capturing manual state as code, no permission change) AND `--force` was passed.

Commits: `eng-infra:MIT-3467-sonarqube` `de1849a` + `1348e26`; `mithril:MIT-3467` `5a72c5f64`.

**4. cicd-agent image grew Node.js + a second JDK**:

T07/F19b assumed one JDK (25). When the per-module toolchain in backend was exercised, `keycloak-hybrid-pq-provider` requested JDK 17 (`keycloakJavaVersion=17`, Keycloak 26 SPI compat). Image had to carry both. Then wallet's `npm install` for type resolution required Node — alpine/k8s base ships only kubectl/helm/etc.

Final image content (`cicd-agent:sonar-jdk17-25-bef7e41`, digest `sha256:720efbd9...`):
- openjdk25 + openjdk17 (both full JDKs — Gradle toolchain selects by language version)
- nodejs (24.15.0) + npm (11.12.1)
- sonar-scanner CLI 6.2.1.4610
- Plus the existing kubectl/helm/oc/az/ansible

Image is **only** consumed by `sonar-scan.Jenkinsfile`. The five orchestrator pipelines (`deploy*`, `demo-orchestrator`, `qa-orchestrator`, `staging-orchestrator`) keep using `cicd-agent:latest` (still the JRE-17 build, unchanged from T11 retag). FINDING §F19b's split-image strategy holds.

**5. SHA-pinning the agent image (not the sliding tag)**:

K8s default `imagePullPolicy=IfNotPresent` for non-`:latest` tags caches layers per-node forever. When we rebuilt the image and updated only the sliding `:sonar-jdk17-25` tag, the next build's pod kept using the old digest. Solution: pin Jenkinsfile to the immutable `:sonar-jdk17-25-<eng-infra sha>` tag. Each image rebuild now requires a one-line Jenkinsfile bump, but the deployed image is traceable to an eng-infra commit and rollback is `git revert`.

Alternative considered but rejected: `imagePullPolicy: Always`. Works but adds a registry round-trip on every job, and the SHA pin is more auditable.

## F20. Existing root `sonar-project.properties` will be deleted

**What**: Today's repo-root `sonar-project.properties` sets `sonar.projectKey=mithril` and was authored for the legacy external Sonar (`dsocodescan.cbuae.gov`). After T08 deploys the new Sonar, the legacy file is **stale** — it points at the old server's project name and listed sources are partial (12 dirs vs the 16-module reality, see F22).

**Why it matters**: Per-module scan integration is a follow-up. Keeping the legacy file around during the gap creates confusion (which Sonar? which project key?). T12 deletes the file.

**How we handle it**: T12 deletes `sonar-project.properties` at repo root. No replacement is added in this rollout. When per-module integration is later picked up, `backend/`, `portal/`, `wallet/` each get their own properties file.

**Do NOT**: re-introduce a root `sonar-project.properties` — the per-module pattern is the target.

## F21. Legacy GHA workflow `trigger-sonar-analysis.yaml` is superseded

**What**: `.github/workflows/trigger-sonar-analysis.yaml` triggers a Jenkins job that scans against the legacy external Sonar. After T08 the new server is the canonical Sonar; the legacy workflow's webhook (`mithril-sonar-trigger-token`) and target server (`dsocodescan.cbuae.gov`) are both obsolete for this codebase.

**Why it matters**: Leaving it in place will continue to publish to the legacy `mithril` project, confusing future dashboard consumers.

**How we handle it**: T13 deletes the workflow. Before deletion, sweep for any caller (`grep -rn "trigger-sonar-analysis\|mithril-sonar-trigger-token" .`) and migrate or warn. The Jenkins JCasC for the **old** `sonar-analysis` job can be removed independently once no callers remain — cross-repo, out of scope for this rollout.

## F22. Backend has 16 src/main modules + 15 src/test modules (kept for the per-module follow-up)

**What**: The existing root `sonar-project.properties` enumerates only 12 source dirs and 10 test dirs for backend. It is **incomplete** — it omits `cbdc-api`, `cbdc-integration`, `keycloak-hybrid-pq-provider`, `keycloak-migration`. Cross-check against `backend/settings.gradle` and the filesystem confirmed: 16 modules have `src/main`, 15 have `src/test` (`notification` and `pqc-keygen` are source-only).

**Why it matters**: Following the old root config blindly during the per-module follow-up would under-report backend code.

**How we handle it**: When `backend/sonar-project.properties` is later authored (follow-up), list all 16 modules in `sonar.sources` and all 15 test modules in `sonar.tests`. Re-verify at integration time in case `settings.gradle` changes.

## F23. Helm chart version pinning matters

**What**: The pasted plan suggests `sonarqube_helm_version: "10.8.0"`, but advises verifying via `helm search repo sonarqube/sonarqube --versions | grep 10.8` before apply. The chart version controls schema for `account.adminPasswordSecretName` and friends — older charts use `account.*` keys, newer charts (post-10.x) deprecated those for `setAdminPassword` or other mechanisms.

**Why it matters**: T05 (Helm values) and T06 (playbook) reference key names that depend on chart version. A wrong version pin produces silent mis-config — Sonar boots with default `admin/admin` even though a Secret exists.

**How we handle it**: Before T08 deploy, run `helm show values sonarqube/sonarqube --version <pinned>` and confirm the admin-secret key shape matches what's in T05's values file. If the chart's keys have moved, either bump the pin or adapt the values.

## Template for new findings

```
## F24. Jenkinsfile `parameters {}` block overrides the JCasC seed's params on every run after the first

**What**: When a Jenkins Pipeline job is provisioned by the Job DSL (JCasC `pipelineJob {...}`), the JCasC `parameters` block sets the *initial* parameter set. But the Jenkinsfile's own `parameters {...}` block is authoritative once the pipeline runs — Jenkins reconciles the parameter set to whatever the Jenkinsfile declares. Any parameter the JCasC defines but the Jenkinsfile omits is **erased** on the first run.

**Why it matters**: Surfaced in T16 phase 1 iteration. `sonar-pr.Jenkinsfile`'s parameters block originally declared just `COMPONENT, COMMIT_SHA, PR_NUMBER, PR_BRANCH`. JCasC for that job also declared `PIPELINE_BRANCH=MIT-3467` (used by the SCM checkout to load the Jenkinsfile itself). After the first run, the `PIPELINE_BRANCH` parameter vanished from the job, and the next GHA-triggered run had no PIPELINE_BRANCH — SCM defaulted to whatever branch Jenkins guesses (often `main`), where the Jenkinsfile didn't exist yet. The smoke test from T15 only worked because that *was* the first run after seed.

Same class of bug latent in `sonar-scan.Jenkinsfile`: its `parameters {}` declared `PIPELINE_BRANCH` but with default `'main'`. JCasC default was `'MIT-3467'`. The Jenkinsfile's `'main'` won on every run after the first — so the dispatched sonar-scan kept trying to load itself from `main`.

**How we handle it**:
- Every parameter the JCasC declares MUST also be declared in the Jenkinsfile's `parameters {}` block, with the **same default**.
- During the transitional MIT-3467 → main rollout, both keep `PIPELINE_BRANCH` defaulted to `'MIT-3467'`. After merge to main, both flip to `'main'` in one follow-up commit.
- The wrapper-dispatcher pattern (sonar-pr calls sonar-scan via `build job:`) should **explicitly forward** every parameter — including PIPELINE_BRANCH — instead of relying on the called job's defaults.

**Do NOT**: declare parameters only in JCasC. The Jenkinsfile is the source of truth at runtime; if a parameter is missing from `parameters {}` the JCasC default exists for exactly one run and then disappears.

## F25. Jenkins in SAML-only mode — API tokens must be generated via SAML-authenticated UI

**What**: This Jenkins is deployed with `securityRealm: saml` (Entra ID SAML SSO). The chart's `controller.admin.password` / K8s `jenkins/jenkins-admin-password` Secret is created during install but **is not usable** for API auth once JCasC replaces the SecurityRealm with SAML. There is no local-password fallback (the `local-security-realm` block in `values.yaml` is intentionally commented out).

**Why it matters**: T16 needs a Jenkins API token for GHA to poll build status. The natural CLI path (`POST /me/descriptorByName/.../generateNewToken` with the chart-stored admin password) fails with HTTP 401, because basic auth is rejected unconditionally. There is no admin-bypass for SAML in this configuration.

**How we handle it**:
- API tokens are generated by a human, through the Jenkins UI, after a SAML browser login. Path: Jenkins UI → click name top-right → Configure → API Tokens → Add new Token. Token is tied to the SAML user's email (lowercased) — that's the value for `JENKINS_API_USER`.
- A dedicated "service-account" identity for the token would need an Entra non-human SAML user provisioned by an admin; out of scope for T16.
- Token can still be revoked from the same UI page if compromised. Rotation policy is per-token, not centrally managed.

**Do NOT**:
- Don't try to use the K8s-stored `jenkins-admin-password` for any API auth — it's defunct in this configuration. The Helm chart writes it but it's never connected to a working SecurityRealm.
- Don't enable the `local-security-realm` JCasC block thinking it'll restore the admin password — it would, but the SAML realm wins (last-wins in JCasC `securityRealm`), so the local block is just dead config.
- Don't dump the K8s secret thinking "ah this is the answer" (we did this for ~10 min during T16; documented here so the next agent doesn't repeat it).

## F26. CF Access service tokens are bound per-application, not org-wide

**What**: A Cloudflare Zero Trust service token (Client ID + Client Secret) is just a credential — it has no inherent access to anything. Access rights are granted by **per-application policies** with `decision: non_identity` that `include: service_token`. The same service token only reaches applications whose policies explicitly include it.

**Why it matters**: Surfaced during T16 phase 2. The existing `github-actions-jenkins-webhook` service token was authorized only on the `jenkins.takamul.cc` Access app (resource `cloudflare_zero_trust_access_policy.cicd_service_token`). When the GHA composite action started calling `sonarqube.takamul.cc/api/qualitygates/...` with the same headers, Cloudflare returned a JSON-shaped 403:

```json
{"message":"Forbidden. You don't have permission to view this.…",
 "status_code":403,"aud":"…","ray_id":"…","mtls_stat":…}
```

The `ray_id` + `aud` + `mtls_stat` fields are CF Access telemetry — meaning CF rejected the request before it ever reached Sonar. Sonar itself never saw the call.

**How we handle it**: Add a parallel `cloudflare_zero_trust_access_policy` resource for every Access app the service token needs to reach. T16 added `cicd_service_token_sonar` pointing at the sonarqube app, including the same `cloudflare_zero_trust_access_service_token.github_actions.id`. Applied scoped (`-target=cloudflare_zero_trust_access_policy.cicd_service_token_sonar`) to avoid touching unrelated cloudflare-module drift.

If multiple apps need the same token in the future, refactoring to a `for_each` over a list of app keys would be cleaner — kept as a parallel resource here because there are still only two apps.

**Do NOT**:
- Don't assume a CF Access service token works against any internal app. It works only where a policy says so.
- Don't try to "edit the service token" to add applications — service tokens are credentials. The applications are what get edited.

## F27. npm 11 enforces peer-dependency conflicts as errors by default

**What**: npm 11 changed default behavior to fail on unresolvable peer-dep conflicts (previously a warning). The error code is `ERESOLVE`. Wallet's `package.json` has at least one such conflict (`react-native-screens@4.25.0` wants `react-native >= 0.82.0`; project pins `0.80.0`).

**Why it matters**: cicd-agent image carries Node 24 / npm 11 (FINDING §F19c). The T16 wallet sonar-scan stage calls `npm install` to populate `node_modules` for Sonar's TS analyzer. Without a flag, npm 11 aborts with `npm error code ERESOLVE` and the Jenkins build fails.

**How we handle it**: Pass `--legacy-peer-deps` to the npm install in `sonar-scan.Jenkinsfile`'s wallet stage. This recovers npm 10-era behavior — peer-dep conflicts become warnings, install proceeds, `node_modules` populates. Acceptable because: (a) the npm install is only for Sonar type resolution, not for shipping an actual build; (b) npm's own error message suggests this exact flag as the remediation.

**Do NOT**:
- Don't fix the underlying conflict in this rollout — it's wallet-team work, may involve bumping `react-native` or downgrading sibling packages, and risks breaking real RN builds.
- Don't add `--force` instead — that hides MORE classes of errors than `--legacy-peer-deps` and risks corrupting `node_modules`.
- Don't remove the flag once wallet's deps are aligned without explicit verification by the wallet team; the flag is harmless when no conflicts exist.

## F28. Concurrent PR scans race on shared Sonar project keys (Community Build)

**What**: With `pr-branch` sourced from `pull_request.base.ref`, all PRs into the same base branch publish to one Sonar project (e.g. every PR into `main` writes to `mithril-${component}-main`). Sonar Community Build has no native branch/PR analysis — it stores one set of measures per project, overwritten each scan. Sonar's Compute Engine serializes processing per project, so no data corruption — but the final state reflects whichever analysis was processed last.

**Why it matters**: For a team that opens many PRs per day (50+ in mithril's case), three failure modes show up:

1. **Stale dashboard**: clicking the Sonar dashboard link from PR-A's GitHub Check shows metrics from whichever PR finished most recently — not PR-A.
2. **Wrong-PR inline comments**: the composite action queries `/api/issues/search?sinceLeakPeriod=true` after polling its own Jenkins build. If a different PR's analysis landed in Sonar between Jenkins finishing and the API read, the issues returned reflect that other PR's state. Inline review comments on PR-A might highlight issues introduced by PR-C.
3. **Wrong-PR Check metrics**: same race for `/api/measures/component` — the metrics table in PR-A's Check could be PR-C's numbers.

**How we handle it**: Race is **accepted** in the current rollout. The GitHub Check's pass/fail conclusion is still correct (it comes from Jenkins via `sonar.qualitygate.wait=true`, which polls the analysis Sonar processed FROM OUR SCAN — Jenkins won't exit until its own analysis is processed). Only the *enrichment* numbers race.

Documented in:
- `task/T16.md` "Known limitations" section
- `task/REUSE.md` step 5 note for adopting repos
- `.github/workflows/sonar-scan-pr.yml` concurrency-group comment

**Mitigation paths if it becomes painful**:
- **Preferred — adopt `sonarqube-community-branch-plugin`** (<https://github.com/mc1arke/sonarqube-community-branch-plugin>): third-party Java agent that adds native branch + PR analysis to Community Build. Release cadence matches Sonar's; `26.4.0` is compatible with our `26.4.0.121862` deployment. Helm-installable via `plugins.install` + `sonar.web.javaAdditionalOpts` / `sonar.ce.javaAdditionalOpts`. Replaces our 6-project model with 3 projects each carrying branch/PR scopes inside; scanner calls switch from project-key suffixing to `sonar.branch.name=...` / `sonar.pullrequest.{key,branch,base}=...` properties. **Eliminates F28 entirely.** ~5 hours of work. Risk: third-party, not SonarSource-supported (their warning is mainly about migrating back to commercial editions — moot for a team that's committed to Community Build). Tracked as a T17 candidate in STATE.md.
- **Per-PR project keys** + cleanup hook: project key = `${prefix}-${component}-pr${pr_number}`. Each PR has its own isolated Sonar project. On PR close, GHA workflow calls `POST /api/projects/delete` to remove the project. Adds complexity (cleanup workflow), introduces project sprawl during PR lifetime (~200-300 transient projects at peak for a high-volume team). Estimated 1 day to implement + validate. Inferior to the plugin path — extra moving parts for the same outcome.
- **Sonar Developer Edition upgrade**: native PR analysis. License cost (~$1-3K/year for 60KLOC scale). Replaces T08 helm install (different chart, different image tag). Architecturally clean — Sonar handles isolation natively. Only path that's SonarSource-supported.

**Do NOT**:
- Don't add GHA-level concurrency to serialize PR scans per project (`group: sonar-${component}-${base}`). At 50 PRs/day with backend at ~12min/scan, queue depth would reach hours. Reviewers would never see Checks complete during business hours.
- Don't trust the metric numbers in a Check posted during a high-traffic window — the pass/fail is still right, the numbers may not be.

## F<next>. <short title>

**What**: <observation>

**Why it matters**: <impact>

**How we handle it**: <the decision>

**Do NOT**: <tempting wrong path, if any>
```
