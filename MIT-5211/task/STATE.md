# STATE — MIT-5211 APISIX on Silmarils Lower-AKS

> Read this first when resuming. Live operational status.

## Pickup steps for fresh context

1. Read `PLAN.md` for the overall direction. The plan is frozen as of
   2026-05-15; deviations must be captured here in STATE.md and/or in a
   numbered finding in `FINDING.md` (not yet created).
2. Skim the **Pre-T01 gates** table below — implementation does not
   start until all eight are green.
3. The local-only source of truth lives in `silmarils/apisix/` of the
   silmarils repo (assets: `apisix.yaml.template`, `config.yaml`,
   `plugins/cert-validator.lua`, `plugins/org-router.lua`,
   `org_routing.json`, 17 `.p12` client certs, `qa-tests/`). All
   ansible work must preserve the plugin contract — header is
   `X-Forwarded-Client-Cert`, CA bundle at
   `/usr/local/apisix/certs/ca.pem`.
4. Sibling work lives in `takamulai/eng-infra`
   (`cloudflare/terraform.tfvars` for the tunnel ingress rule) and
   eventually in mithril (none planned this PR — APISIX-on-OKD is
   referenced for shape only).

## Current state (2026-05-15)

| Task | Spec | Status | Notes |
|---|---|---|---|
| `PLAN.md` (frozen) | — | done | Copied verbatim from working `plan.md` on 2026-05-15. |
| Pre-T01 gates | `T01.md` | **pending — all 8** | See gate table below; blocks T02–T07. |
| T02 — Custom image + ACR pipeline | `T02.md` | pending | Blocked on T01 (G1, G2). |
| T03 — Layer 15a base | `T03.md` | pending | Blocked on T01 (G2, G3, G4), T02. |
| T04 — Layer 15b TLS + data plane | `T04.md` | pending | Blocked on T01 (G5), T03. |
| T05 — Production hardening (NP/PDB/HPA) | `T05.md` | pending | Blocked on T01 (G7), T04. |
| T06 — Wiring (`kubernetes.yaml` + `variables-qa.yaml`) | `T06.md` | pending | Blocked on T01 (G8), T05. |
| T07 — Cloudflare Tunnel sibling PR (eng-infra) | `T07.md` | pending | Blocked on T01 (G6); parallelisable with T03–T06. |
| T08 — QA smoke + E2E verification | `T08.md` | pending | Blocked on T06, T07. |

## In-flight task

None — pre-implementation phase. The current obligation is **T01**:
walk the eight pre-T01 gates and turn each green or document the
deviation. Until T01 closes, T02+ inputs (runtime UID, ACR-pull
strategy, `.p12` source path, `cloudflared` SA, locked org_routing
values) are speculation.

## Pre-T01 gates (must all be green before T01 is authored)

| # | Gate | How to verify | Status |
|---|---|---|---|
| G1 | AKS managed-identity has `AcrPull` on `silmarilsacr` | `az aks show -g <rg> -n <cluster> --query identityProfile.kubeletidentity` then `az role assignment list --assignee <kubelet-id-objectId> --scope <acr-resource-id>` | pending |
| G2 | Upstream `apache/apisix:3.11.0-debian` runtime UID | `docker run --rm --entrypoint id apache/apisix:3.11.0-debian` | pending |
| G3 | AKS cluster version ≥ 1.27 (PSA stable) | `kubectl version --short` against `silmarils-qa` context | pending |
| G4 | PSA `restricted` compatibility of `apache/apisix:3.11.0-debian` | Trial pod with `runAsNonRoot`, `readOnlyRootFilesystem`, emptyDir at `/usr/local/apisix/logs`, subPath CM mounts at writable `conf/` paths | pending |
| G5 | `.p12` relocation policy | Decide: relocate to `iac/eng-infra/shared-k8s/ansible/files/silmarils/apisix-client-certs/`, or symlink, or `lookup('fileglob', ...)` reaching back into `apisix/client-certs/` | pending |
| G6 | Cloudflare Tunnel sibling PR filed against `takamulai/eng-infra` | PR adds one `silmarils_ingress_rules` entry: `silmarils-qa-apisix.takamul.cc` → `https://apisix.apisix-silmarils.svc.cluster.local:19888` | pending |
| G7 | `cloudflared` SA name + namespace | `kubectl -n <cf-namespace> get deploy cloudflared -o jsonpath='{.spec.template.spec.serviceAccountName}'` in silmarils AKS | pending |
| G8 | QA `org_routing` upstream targets | Confirm with platform/silmarils owners: `jisr-simulator.silmarils-qa.svc.cluster.local:8080`, `mbridge-mock.silmarils-qa.svc.cluster.local:8080`, etc. — lock all 17 bank → upstream entries before authoring `variables-qa.yaml` | pending |

## Open questions / pending decisions

- **`.p12` source path inside ansible** (gate G5) — verbatim copy under
  the ansible `files/` tree, a relative symlink from there back to
  `apisix/client-certs/`, or `community.crypto.openssl_pkcs12` with an
  absolute repo-relative path. Affects whether 15b is reusable in
  isolation (e.g. for a future dev environment).
- **Which 17 bank → upstream mappings are real in QA** (gate G8) —
  every CN in `cn_whitelist` needs a corresponding `org_routing` entry
  pointing to either an in-cluster simulator or a real LFI endpoint;
  mismatches will surface as 503 from `org-router.lua`.
- **Whether the `silmarils-dev` Cloudflare tunnel is shared across all
  silmarils AKS namespaces or scoped per-app** — answer determines
  whether we can reuse it for `silmarils-qa-apisix.takamul.cc` or need a
  new tunnel. Plan currently assumes reuse (`eng-infra/cloudflare/main.tf:450`).
- **`silmarils/ansible-deploy` Jenkins pipeline** — F1 follow-up. Until
  it lands, deploys are manual via `ansible-playbook ... --tags
  apisix-base,apisix-silmarils-lfi`. Operator must have
  `silmarils-aks-admin` kubeconfig context.

## Recent changes

- `2026-05-15` — Ticket folder created. `PLAN.md` frozen from the
  working `plan.md`. `STATE.md` initialised with the eight pre-T01
  gates as the current blocker.
- `2026-05-15` — Task fan-out drafted: `T01.md` (gate verification)
  through `T08.md` (QA smoke + E2E) authored as frozen specs. T01 is
  the next live task; T02–T08 are pending its closure (or documented
  deviation).
