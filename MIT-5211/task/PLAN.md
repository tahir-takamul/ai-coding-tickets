# Plan — APISIX on Silmarils Lower-AKS (Two-Layer Ansible Deployment)

> Master plan for MIT-5211. Live state lives in `STATE.md`; gotchas in
> `FINDING.md` (created on first finding). Per-task specs are `T01.md`,
> `T02.md`, … and are frozen once accepted.
>
> Frozen from the working plan on 2026-05-15. Body below is verbatim.

---

# APISIX on Silmarils Lower-AKS — Two-Layer Ansible Deployment

## Context

Silmarils developers maintain a fully working Apache APISIX gateway under `apisix/` at the repo root, used today only via `docker-compose` for local testing. Its assets (`apisix.yaml.template`, `config.yaml`, custom Lua plugins `cert-validator.lua` + `org-router.lua`, `org_routing.json`, 17 bank `.p12` client certs, an E2E test harness) form the single source of truth for how silmarils wants to route external LFI (bank) traffic — mTLS termination, CN-to-LFI-id mapping, dynamic upstream selection per LFI, and forward-auth back to `dc-tang`.

This plan promotes that local-only setup into silmarils' AKS **QA** environment (`silmarils-qa`) without rewriting the routing logic. Dev is not in scope (no `silmarils-dev` target today); the same playbooks will be reused for dev later by adding `variables-dev.yaml` overrides — the design supports it but does not deploy it here. The intent (per user direction) is two ansible layers:

- A **generic APISIX runtime install** (reusable primitive — namespace, workload, internal service).
- A **silmarils LFI overlay** that supplies the actual routes, plugin configuration, mTLS material.

Internal portal/backend traffic continues through the existing NGINX ingress unchanged. APISIX is the LFI edge.

**QA-AKS exposure model (locked):** Cloudflare Zero Trust Tunnel (`silmarils-dev` tunnel — name is historical, scopes the whole silmarils AKS cluster — managed in `eng-infra/cloudflare/main.tf:450`) terminates TLS at the Cloudflare edge and forwards HTTPS to in-cluster Services. APISIX is exposed via one new tunnel ingress rule for QA pointing at its ClusterIP Service. There is no public Azure LoadBalancer, no NSG bank-IP allowlist, and no Cloudflare edge-mTLS for this rollout — **APISIX itself handles all mTLS decisions** (same as the local docker-compose stack). Test/CI traffic presents bank `.p12` certs via the `X-Forwarded-Client-Cert` header, exactly as `apisix/qa-tests/run-passthrough-tests.sh` does today. On-prem (F5 fronting APISIX) is a follow-up rollout.

The mithril APISIX-on-OKD playbook is referenced for *shape* only — silmarils diverges in mode (standalone vs full), plugins (custom Lua vs none), secret source (cert-manager vs Vault), and exposure (Cloudflare Tunnel vs OKD Router/F5).

## Architectural decisions (locked)

| Concern | Decision | Rationale |
|---|---|---|
| Mode | **Standalone** (`config_provider: yaml`; no etcd, no apisix-ingress-controller) | Source-of-truth alignment with local dev; 8 stable routes don't need CRD/GitOps churn; smaller surface. |
| Image | **Custom rootless image baked with Lua plugins**: `silmarilsacr.azurecr.io/silmarils-apisix:<tag>` from `apache/apisix:3.11.0-debian` | Plugins versioned with the image; immutable artefact; fast pod start; no ConfigMap reload coupling for plugin code. |
| Lua plugin paths in image | `cert-validator.lua` → `/usr/local/apisix/apisix/plugins/`; `org-router.lua` → `/opt/apisix-custom/apisix/plugins/` (matches `extra_lua_path` in `config.yaml`) | Built-in plugins live in core tree; vendor plugins on separate search path survive APISIX upgrades. |
| Routes/config delivery | **ConfigMap-mounted** (`apisix.yaml`, `config.yaml`, `org_routing.json`) | Env-specific; not appropriate to bake into image. |
| Server TLS | **cert-manager** Certificate CR using existing `silmarils-ca-issuer` (`04-ca-certificates.yaml`) | Matches existing silmarils pattern (`dc-tang-tls`); zero new infra. Internal CA is fine because Cloudflare Tunnel uses `origin_no_tls_verify` to origin. |
| Client mTLS truststore (bank CA bundle) | **Ansible-assembled K8s Secret** from `apisix/client-certs/*.p12` via `community.crypto.openssl_pkcs12` | Reuses repo-checked certs; mounted into APISIX at `/usr/local/apisix/certs/ca.pem` (same path the plugin expects today). |
| mTLS termination | **APISIX itself** (reads `X-Forwarded-Client-Cert` header — unchanged from local) | Cloudflare Tunnel does not pass through raw TLS; it terminates at edge and re-establishes HTTPS to origin. Bank certs in dev/qa are presented by test harnesses via the header (matches local docker behavior). On-prem (out of scope) will have F5 inject the same header. |
| External exposure | **Cloudflare Zero Trust Tunnel** (existing `silmarils-dev` tunnel; new ingress rule per env) | Silmarils convention. No public LB. No Azure NSG/source-ranges. |
| APISIX Service type | **ClusterIP only** | Cloudflare Tunnel reaches it cluster-internally via `https://apisix.apisix-silmarils.svc.cluster.local:19888`. |
| Secret strategy | **No Azure Key Vault** | Silmarils has zero KV patterns today; introducing one is scope. Defer. |
| Manifest delivery | **Inline `kubernetes.core.k8s` with Jinja templates** | Silmarils convention (NGINX + cert-manager are the only Helm exceptions). |

## Layer architecture (the two playbooks)

**Layer split principle:** 15a owns *workload shape and lifecycle contract*; 15b owns *data*. The Deployment is in 15a but starts **paused**; 15b populates ConfigMaps/Secrets, computes a checksum annotation on the pod template, and unpauses. A future `15c-apisix-portal-lfi.yaml` could add new traffic classes without modifying 15a or 15b.

### 15a — `15a-apisix.yaml` (base APISIX runtime)

1. Namespace `apisix-silmarils` with PSA label `pod-security.kubernetes.io/enforce: restricted` (silmarils' first PSA-enforced namespace; AKS ≥ 1.27 required).
2. ServiceAccount `apisix` with `automountServiceAccountToken: false` (standalone mode does not call the K8s API).
3. ConfigMap `apisix-daemon-config` wrapping `config.yaml` (daemon-level config: ports, registered plugins, prometheus toggle). Rendered from `templates/apisix-daemon-config.yaml.j2`.
4. Deployment `apisix` with `spec.paused: true` on first apply. Mounts (by well-known name) `apisix-routes`, `apisix-org-routing`, `apisix-daemon-config`, `apisix-server-tls`, `apisix-ca-bundle`. Image, probes, securityContext, resources, volumes defined here.
5. ClusterIP Service `apisix` (ports 19880 admin/debug, 19888 gateway).

15a is intentionally not usable on its own — without 15b, the Deployment remains paused. This is correct layering.

### 15b — `15b-apisix-silmarils-lfi.yaml` (silmarils overlay)

1. cert-manager `Certificate` CR (issuer `silmarils-ca-issuer`, secretName `apisix-server-tls`, DNS names: cluster-internal `apisix.apisix-silmarils.svc.cluster.local` + the Cloudflare-fronted hostname `silmarils-{env}-apisix.takamul.cc`).
2. Poll for Secret `apisix-server-tls` to materialize (`k8s_info` with `until` clause, 5-min timeout; dump cert-manager Challenge/Order on failure).
3. Assemble CA bundle from `.p12` files in `iac/eng-infra/shared-k8s/ansible/files/silmarils/apisix-client-certs/` using `community.crypto.openssl_pkcs12`; apply Secret `apisix-ca-bundle` (single `ca.pem` key concatenating all CA chains).
4. Read materialized server cert+key via `k8s_info`; jinja-render the route ConfigMap `apisix-routes` (a K8s-flavored variant of `apisix/apisix.yaml.template` with no `__SSL_CERT__`/`__SSL_KEY__` placeholders — direct jinja insertion of PEMs read from `apisix-server-tls`, loops over `silmarils.apisix.cn_whitelist` for the cert-validator plugin_config).
5. ConfigMap `apisix-org-routing` from `silmarils.apisix.org_routing` dict.
6. Compute `sha256` over the three ConfigMaps; patch Deployment `apisix` `spec.template.metadata.annotations."checksum/config"` (**on the pod template**, not the Deployment metadata — this is what triggers rollouts on config change) and unpause (`spec.paused: false`).
7. NetworkPolicy (default-deny; allow ingress on 19880/19888 from the `cloudflared` ServiceAccount in its namespace; allow egress to `dc-tang` in `namespaces.silmarils` on 28888 and to LFI upstream IPs/DNS:53).
8. PodDisruptionBudget `minAvailable: 1`.
9. HorizontalPodAutoscaler (CPU target 70%; dev 1–2, qa 2–4).
10. `inc/rollout-wait.yml` for Deployment `apisix`.
11. Smoke step: `kubectl get cm apisix-routes -o jsonpath` (sanity), report tunnel hostname for operator.

## Cloudflare Tunnel ingress (out-of-band PR in `eng-infra`)

A sibling PR to `takamulai/eng-infra` adds one ingress rule to `silmarils_ingress_rules` in `cloudflare/terraform.tfvars`:

```hcl
{
  hostname = "silmarils-qa-apisix.takamul.cc"
  service  = "https://apisix.apisix-silmarils.svc.cluster.local:19888"
},
```

`origin_no_tls_verify: true` (already the default for the tunnel) accommodates the internal-CA server cert. No new Cloudflare resources, no Access policy on this hostname (mTLS is at APISIX, not at the Cloudflare edge).

DNS CNAME to `<tunnel-id>.cfargotunnel.com` is created automatically by the existing `cloudflare_record.silmarils_tunnel_ingress` resource.

A second ingress entry for a future `silmarils-dev-apisix.takamul.cc` will be added when dev comes online — same shape, just another `silmarils_ingress_rules` element.

## Custom image (Dockerfile)

Located at `apisix/Dockerfile` (alongside existing dev assets so they evolve together):

- `FROM apache/apisix:3.11.0-debian`
- `ARG APISIX_VERSION` for tag pinning
- `COPY apisix/plugins/cert-validator.lua /usr/local/apisix/apisix/plugins/cert-validator.lua`
- `COPY apisix/plugins/org-router.lua /opt/apisix-custom/apisix/plugins/org-router.lua`
- `chown` to the upstream image's runtime UID (verify before T01: upstream `apache/apisix:3.11.0-debian` runs as a specific UID — pin via `USER` directive after confirming)
- `.dockerignore` excludes `client-certs/`, `certs/`, `qa-tests/`, `*.p12`

Build pipeline: new GitHub Actions workflow `.github/workflows/apisix-image.yaml`, triggered on push to `main` when `apisix/**` changes, using silmarils' existing ACR OIDC federated identity (matches `mbridge-mock-build-push-acr.yml` pattern). Tag = `git describe --tags --always`.

## Plugin behavior — unchanged from local

`cert-validator.lua` and `org-router.lua` ship as-is. The plugin contract:

- Reads `X-Forwarded-Client-Cert` (header injected by test harness or, eventually, by F5 on-prem)
- Validates against CA bundle at `/usr/local/apisix/certs/ca.pem` (mounted from Secret `apisix-ca-bundle`)
- Resolves CN → lfi-id via the `cn_whitelist` map in the route plugin_config (rendered into `apisix.yaml` from `silmarils.apisix.cn_whitelist`)
- Sets `X-LFI-ID` downstream; strips `X-Forwarded-Client-Cert` to prevent spoofing past APISIX
- `org-router.lua` reads `X-LFI-ID`, looks up `/usr/local/apisix/conf/org_routing.json` (mounted from `apisix-org-routing` ConfigMap), sets upstream

The `qa-tests/run-passthrough-tests.sh` script works against the tunnel-fronted hostname after pointing its base URL at `https://silmarils-{env}-apisix.takamul.cc` and continuing to inject `X-Forwarded-Client-Cert` (Cloudflare Tunnel forwards arbitrary headers to origin transparently).

## Per-env variable schema

Add a `silmarils.apisix` block to `iac/eng-infra/shared-k8s/ansible/variables-qa.yaml` only (dev is not in scope). Minimum surface; cluster-internal concerns stay as template defaults.

```yaml
silmarils:
  apisix:
    enabled: true
    namespace: "apisix-silmarils"
    fqdn: "silmarils-qa-apisix.takamul.cc"   # Cloudflare-fronted hostname (tunnel rule lives in eng-infra)
    image:
      repository: "silmarilsacr.azurecr.io/silmarils-apisix"
      tag: "3.11.0-silmarils.1"
    replicas: 2
    resources:
      requests: { cpu: "200m", memory: "256Mi" }
      limits:   { cpu: "1",    memory: "512Mi" }
    autoscaling:
      enabled: true
      min_replicas: 2
      max_replicas: 4   # qa-sized; revisit when dev/staging come online
      cpu_target: 70
    cn_whitelist:          # CN → lfi-id (mirrors apisix.yaml.template plugin_config 1)
      "system@fab.com":  "lfi-FAB"
      "system@adcb.com": "lfi-ADCB"
      # ... 17 entries
    org_routing:           # lfi-id → upstream (drives org-router.lua)
      lfi-FAB:
        host: "jisr-simulator.silmarils-qa.svc.cluster.local"   # qa: in-cluster sim
        port: 8080
        scheme: "http"
        path_prefix: "/mbridge/ws"
      # ... 17 entries
    dc_tang:               # static upstream for /api/v1/issuance-platform
      host: "dc-tang.silmarils-qa.svc.cluster.local"
      port: 28888
      scheme: "https"
    forward_auth:
      uri: "https://dc-tang.silmarils-qa.svc.cluster.local:28888/auth/inbound"
      ssl_verify: true
```

## Files to create / edit

### New (silmarils repo)

```
apisix/Dockerfile                                                        # custom image
apisix/.dockerignore
apisix/build-push.sh                                                     # local helper; CI calls equivalent

.github/workflows/apisix-image.yaml                                      # GH Actions build/push to ACR

iac/eng-infra/shared-k8s/ansible/playbooks/15a-apisix.yaml               # Layer 1 (base)
iac/eng-infra/shared-k8s/ansible/playbooks/15b-apisix-silmarils-lfi.yaml  # Layer 2 (overlay)

iac/eng-infra/shared-k8s/ansible/templates/apisix-namespace.yaml.j2      # PSA-labeled ns
iac/eng-infra/shared-k8s/ansible/templates/apisix-serviceaccount.yaml.j2
iac/eng-infra/shared-k8s/ansible/templates/apisix-daemon-config.yaml.j2  # config.yaml CM
iac/eng-infra/shared-k8s/ansible/templates/apisix-deployment.yaml.j2     # paused initially; checksum annotation
iac/eng-infra/shared-k8s/ansible/templates/apisix-service.yaml.j2        # ClusterIP only
iac/eng-infra/shared-k8s/ansible/templates/apisix-server-cert.yaml.j2    # cert-manager Certificate CR
iac/eng-infra/shared-k8s/ansible/templates/apisix-routes.yaml.j2         # apisix.yaml CM (K8s flavor; cert/key from rendered Secret)
iac/eng-infra/shared-k8s/ansible/templates/apisix-org-routing.yaml.j2    # org_routing.json CM
iac/eng-infra/shared-k8s/ansible/templates/apisix-ca-bundle.yaml.j2      # CA bundle Secret
iac/eng-infra/shared-k8s/ansible/templates/apisix-networkpolicy.yaml.j2
iac/eng-infra/shared-k8s/ansible/templates/apisix-pdb.yaml.j2
iac/eng-infra/shared-k8s/ansible/templates/apisix-hpa.yaml.j2

iac/eng-infra/shared-k8s/ansible/files/silmarils/apisix-client-certs/*.p12   # relocated from apisix/client-certs/ (or symlinked)
```

### Edit (silmarils repo)

```
iac/eng-infra/shared-k8s/ansible/kubernetes.yaml      # replace placeholder lines 199-200 with two include_tasks blocks (apisix-base, apisix-silmarils-lfi)
iac/eng-infra/shared-k8s/ansible/variables-qa.yaml    # add silmarils.apisix.* block (dev added later when dev env exists)
```

### Edit (eng-infra repo — sibling PR)

```
cloudflare/terraform.tfvars   # add one silmarils_ingress_rules entry for the qa apisix hostname
```

### Reused existing patterns/files (do not duplicate)

- `iac/eng-infra/shared-k8s/ansible/playbooks/inc/preflight.yml`
- `iac/eng-infra/shared-k8s/ansible/playbooks/inc/rollout-wait.yml`
- `iac/eng-infra/shared-k8s/ansible/playbooks/04-ca-certificates.yaml` — `silmarils-ca-issuer` ClusterIssuer (already deployed)
- `iac/eng-infra/shared-k8s/ansible/playbooks/11-silmarils-gateway.yaml` — prototype for inline `kubernetes.core.k8s` Deployment/Service/Secret pattern
- `iac/eng-infra/shared-k8s/ansible/playbooks/14d-mbridge-lfi-connector.yaml` + `templates/lfi-connector-registry-configmap.yaml.j2` — prototype for Jinja-templated ConfigMap via `lookup('template', ...) | from_yaml`

## Production hardening (baked in from day one)

- **NetworkPolicy** default-deny ingress+egress; allow ingress 19880/19888 from `cloudflared` SA only; allow egress to `dc-tang.silmarils-{env}.svc.cluster.local:28888` and to in-cluster simulator Services + DNS:53.
- **PodDisruptionBudget** `minAvailable: 1`.
- **HorizontalPodAutoscaler** v2 (CPU 70%; qa 2–4 replicas).
- **Resource requests + limits** explicit per environment.
- **Probes**: TCP socket on 19880 for readiness + startup (`failureThreshold: 30 × 5s`); a `public-api`-served `/healthz` route in `apisix.yaml` for HTTP liveness.
- **securityContext**: `runAsNonRoot: true`, runtime UID confirmed at T01, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`, `seccompProfile.type: RuntimeDefault`. EmptyDir mounts at `/usr/local/apisix/logs` and (via `subPath` ConfigMap mounts) the writable parts of `/usr/local/apisix/conf`.
- **PSA**: `pod-security.kubernetes.io/enforce: restricted` on the namespace.
- **ServiceAccount** dedicated; `automountServiceAccountToken: false`.
- **ConfigMap reload trap fix**: SHA256 over the three ConfigMaps → `checksum/config` annotation on `spec.template.metadata`. Forces pod rollout on any data change.
- **`X-Forwarded-Client-Cert` integrity**: cert-validator.lua already strips the header before forwarding. No additional middleware needed.

## Deploy flow (end to end)

Operator (manual until ansible Jenkins pipeline lands — see Out of scope):

```bash
ansible-playbook iac/eng-infra/shared-k8s/ansible/kubernetes.yaml \
  --tags "apisix-base,apisix-silmarils-lfi" \
  -e silmarils_environment=qa
```

Order of K8s objects created:
**Namespace → ServiceAccount → ConfigMap(daemon) → Deployment(paused) → Service(ClusterIP) → Certificate CR → Secret(server-tls, materialized by cert-manager) → Secret(ca-bundle) → ConfigMap(routes) → ConfigMap(org-routing) → Deployment patch (checksum + unpause) → NetworkPolicy → PDB → HPA**.

## Verification (QA smoke)

```bash
NS=apisix-silmarils
FQDN=silmarils-qa-apisix.takamul.cc

kubectl -n $NS get pod -l app=apisix -o wide                                # 2/2 Ready
kubectl -n $NS get cm apisix-routes apisix-org-routing apisix-daemon-config # all present
kubectl -n $NS get secret apisix-server-tls apisix-ca-bundle                # both present

# In-cluster smoke (bypasses Cloudflare; uses local-dev plugin contract)
kubectl -n $NS run smoke --rm -it --image=curlimages/curl --restart=Never -- \
  curl -k -X POST "https://apisix.$NS.svc.cluster.local:19888/api/v1/issuance-platform" \
       -H 'Content-Type: application/xml' \
       -H "X-Forwarded-Client-Cert: $(cat /tmp/valid-client.pem | jq -sRr @uri)" \
       --data @sample.xml
   # expect 200 (or 401 from dc-tang if api key missing — both prove APISIX → dc-tang reached)

# Through Cloudflare Tunnel
curl -X POST "https://$FQDN/api/commercial" \
     -H 'Content-Type: application/xml' \
     -H "X-Forwarded-Client-Cert: $(cat /tmp/valid-client.pem | jq -sRr @uri)" \
     --data @sample.xml -w "\nHTTP %{http_code}\n"
   # expect 200; bad CN should yield 403 from cert-validator

# Plugins loaded; no schema errors
kubectl -n $NS logs deploy/apisix --tail=100 | grep -E '(cert-validator|org-router)'

# Full E2E (script base URL pointed at the tunnel hostname)
bash apisix/qa-tests/run-passthrough-tests.sh "$VALID_INBOUND_API_KEY"
```

## Out of scope (this PR)

- **On-prem (SIT/UAT/PROD)** — F5 fronting APISIX, F5-level mTLS forwarding (or APISIX direct mTLS depending on F5 mode), prod NSG, separate cert lifecycle. Separate rollout.
- **Cloudflare edge mTLS** — explicit non-goal; APISIX owns all mTLS in lower env.
- **Dev environment rollout** — `silmarils-dev` does not exist in this AKS today; this playbook deploys QA only. When dev comes online, add a `silmarils.apisix` block to `variables-dev.yaml` and a second `silmarils_ingress_rules` entry in `eng-infra/cloudflare/terraform.tfvars`. No 15a/15b changes needed — the design is env-parameterized.
- **Silmarils ansible-deploy Jenkins pipeline** — `silmarils/deploy` today is image-set-only (`kubectl set image`) and cannot run ansible. A sibling PR to `takamulai/eng-infra` is needed to add a `silmarils/ansible-deploy` JCasC entry. **Until that lands, run the playbook manually** from a workstation with `silmarils-aks-admin` context.
- **Azure Key Vault integration** — silmarils has zero KV patterns today; out of scope.
- **APISIX dashboard** — disabled (matches mithril; standalone mode has no admin API anyway).
- **Per-bank rate-limiting / circuit breakers** — add later if needed.
- **Replacing NGINX ingress** for non-LFI traffic.

## Follow-ups

- **F1** — Add `silmarils/ansible-deploy` Jenkinsfile + JCasC seed in `takamulai/eng-infra/cicd/k8s/jenkins/values.yaml`, then wire 15a/15b in.
- **F2** — On-prem rollout: APISIX behind F5; decide F5-edge-mTLS-forward vs APISIX-direct-mTLS; prod NSG; separate cert lifecycle.
- **F3** — Prometheus scrape: edit existing `16-monitoring.yaml` to add APISIX scrape target + import dashboard JSON.
- **F4** — Mirror E2E tests into `qa/` so the QA Jenkins pipeline runs them automatically.
- **F5** — Externalize Lua plugins to ConfigMap if iteration cadence justifies (currently baked into image for immutability).

## Ready-to-implement gates (must be true before T01 starts)

1. **AKS managed-identity ACR pull verified** — `az aks show --query identityProfile.kubeletidentity` confirms `AcrPull` on `silmarilsacr`. If absent, add an ACR pull-secret sync to 15a (matching `07b-sync-secrets.yaml` pattern).
2. **`apache/apisix:3.11.0-debian` runtime UID confirmed** to lock the `securityContext.runAsUser`.
3. **AKS cluster version ≥ 1.27** (for PSA stable). Confirm `kubectl version`.
4. **PSA `restricted` compatibility** of `apache/apisix:3.11.0-debian` verified end-to-end (runAsNonRoot + readOnlyRootFilesystem + emptyDir mounts at `/usr/local/apisix/logs`).
5. **`.p12` relocation** from `apisix/client-certs/` to `iac/eng-infra/shared-k8s/ansible/files/silmarils/apisix-client-certs/` agreed (or symlink path agreed) — keeps local dev intact while ansible has a deterministic source for CA-bundle extraction.
6. **Cloudflare Tunnel sibling PR** filed against `takamulai/eng-infra` adding the QA `silmarils_ingress_rules` entry; merged before the QA smoke test.
7. **`cloudflared` ServiceAccount name + namespace** identified for the NetworkPolicy ingress rule (read from the current tunnel deployment in silmarils AKS).
8. **QA `org_routing` upstream targets confirmed** — almost certainly the in-cluster simulators (`jisr-simulator.silmarils-qa.svc.cluster.local:8080`, `mbridge-mock.silmarils-qa.svc.cluster.local:8080`), but lock the values before `variables-qa.yaml` is authored.

Once those eight gates are green, implementation proceeds in the file order listed under *Files to create / edit*.
