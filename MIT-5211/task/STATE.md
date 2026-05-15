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
| Pre-T01 gates | `T01.md` | **in progress (3 closed: G2, G5, G7; 1 partial: G8; 4 operator-pending: G1, G3, G4, G6)** | See gate table below. T02/T03 unblocked. T06 blocked on G8. |
| T02 — Custom image + ACR pipeline | `T02.md` | **DONE (authored + build-verified, uncommitted in silmarils)** | Dockerfile + .dockerignore + build-push.sh + GHA workflow. `docker build` clean, 109.58 MB. Plugins at correct paths, UID 636. |
| T03 — Layer 15a base | `T03.md` | **DONE (authored + syntax-checked, uncommitted in silmarils)** | 15a-apisix.yaml + 5 templates. Idempotent re-pause guard added; server-tls mounted at `/usr/local/apisix/certs/server` (F05 — T04 locks PEM-source approach). |
| T04 — Layer 15b TLS + data plane | `T04.md` | **next** | Blocked on T01 G5 decision (defaulted to copy), T03 (done). F05 locks server-tls mount path; F03 not yet blocking (T04 doesn't transcribe CN list). |
| T05 — Production hardening (NP/PDB/HPA) | `T05.md` | pending | Blocked on T01 G7 (closed), T04. NetworkPolicy selector revised per F01. |
| T06 — Wiring (`kubernetes.yaml` + `variables-qa.yaml`) | `T06.md` | pending | Blocked on T01 G8 + F03 (owner sign-off), T05. F02 confirms insert-into-gap (not replace). |
| T07 — Cloudflare Tunnel sibling PR (eng-infra) | `T07.md` | pending | Eng-infra workspace confirmed local. Parallelisable with T04/T05. |
| T08 — QA smoke + E2E verification | `T08.md` | pending | Blocked on T06, T07. |

## In-flight task

**T04** is next — Layer 15b overlay (cert-manager Certificate CR, CA
bundle Secret from `.p12`s, routes CM, org-routing CM, checksum+unpause
patch, rollout-wait). Blocked only on G5 policy (defaulted: copy `.p12`
to ansible files dir) and F03 (NBF/EIB CN swap awaits owner — but T04
doesn't transcribe the CN list yet; F03 only blocks T06).

**T07** can run in parallel — small edit in
`/Users/mohd.tahir/DevWorkspace/apisix/eng-infra/cloudflare/terraform.tfvars`
adding one `silmarils_ingress_rules` entry. No agent needed.

**Operator-side outstanding (not blocking authoring)**:
- G1 — AcrPull on silmarilsacr
- G3 — AKS version ≥ 1.27
- G4 — PSA-restricted compatibility (verifiable post-apply)

**Owner-side outstanding (blocks T06)**:
- G8 — 17 lfi-id → QA upstream pairs
- F03 — confirm NBF/EIB CN→lfi-id swap is intentional or fix in source

## Pre-T01 gates (T01 in progress; updated 2026-05-15)

| # | Gate | Status | Result / Notes |
|---|---|---|---|
| G1 | AKS managed-identity has `AcrPull` on `silmarilsacr` | **pending — operator** | Needs `az` + `kubectl` against silmarils-qa. Commands prepared in `T01.md`. Not blocking T02 authoring; affects T03 only (whether to add pull-secret sync). |
| G2 | Upstream `apache/apisix:3.11.0-debian` runtime UID | **✅ green** | UID=636 (apisix), GID=636 (apisix). Verified locally via Docker. Output archived: `logs/T01-G2-runtime-uid.txt`. Locks `securityContext.runAsUser/runAsGroup: 636` (T03) and Dockerfile `USER 636` (T02). |
| G3 | AKS cluster version ≥ 1.27 (PSA stable) | **pending — operator** | Needs `kubectl version` against silmarils-qa. Blocks T03 acceptance only (apply step); does not block authoring. |
| G4 | PSA `restricted` compatibility of `apache/apisix:3.11.0-debian` | **pending — operator** | Trial pod manifest authored as part of T03's deployment template; ad-hoc validation deferred to operator after T03 apply. |
| G5 | `.p12` relocation policy | **✅ decided (default)** | Policy = copy to `iac/eng-infra/shared-k8s/ansible/files/silmarils/apisix-client-certs/`. 17 `.p12` files confirmed in `silmarils/apisix/client-certs/` matching the 17 CN whitelist entries. Actual copy performed as a sub-step of T04 (not now — keeps T01 read-only). User can override before T04 starts. |
| G6 | Cloudflare Tunnel sibling PR (eng-infra) | **pending — T07** | Local `eng-infra/cloudflare/` workspace confirmed at `/Users/mohd.tahir/DevWorkspace/apisix/eng-infra/cloudflare/` (terraform.tfvars present). T07 authors the diff locally; user opens PR. |
| G7 | `cloudflared` SA name + namespace | **✅ closed from repo** | Namespace: `ingress-nginx` (from `variables-qa.yaml:26` → `namespaces.ingress_nginx`). No dedicated SA — pods inherit `default` SA. Pod label: `app: cloudflared`. **Filed as FINDING F01** — NetworkPolicy must select by `namespaceSelector + podSelector`, not by ServiceAccount. T05 updated. |
| G8 | QA `org_routing` upstream targets | **partial — pending owner** | 17 CN→lfi-id pairs locked from `apisix.yaml.template`. **FINDING F03** flags an apparent NBF/EIB swap that needs owner sign-off before transcribing to `variables-qa.yaml` (T06). 17 lfi-id→upstream pairs for QA still need confirmation (likely in-cluster simulators `jisr-simulator.silmarils-qa.svc:8080`, `mbridge-mock.silmarils-qa.svc:8080`, but unconfirmed). |

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
- `2026-05-15` — T01 first pass: G2 verified (UID=636 via local Docker;
  archived to `logs/T01-G2-runtime-uid.txt`), G5 defaulted (copy `.p12`
  to ansible files dir), G7 closed from repo (`ingress-nginx` ns + pod
  label `app=cloudflared`, no SA — see F01). Operator-pending gates
  G1/G3/G4 have prepared commands in T01.md. Findings filed:
  F01 (cloudflared selector), F02 (kubernetes.yaml placeholder shape),
  F03 (NBF/EIB CN→lfi-id swap awaiting owner confirmation),
  F04 (dc-tang:28888 port cross-confirmed).
- `2026-05-15` — **T02 DONE** (agent + verified). 4 files in
  `silmarils/`: `apisix/Dockerfile`, `apisix/.dockerignore`,
  `apisix/build-push.sh`, `.github/workflows/apisix-image.yaml`.
  Local `docker build` clean → 109.58 MB image; plugins land at correct
  paths, container runs as UID 636. Workflow mirrors mbridge-mock OIDC
  + ACR push using `docker/build-push-action@v6`. Silmarils-side
  uncommitted (per project rule — silmarils PR goes up after T04-T08).
- `2026-05-15` — **T03 DONE** (agent + verified). 6 files in
  `silmarils/iac/eng-infra/shared-k8s/ansible/`:
  `playbooks/15a-apisix.yaml` + 5 templates (namespace, SA,
  daemon-config CM, deployment, service). `ansible-playbook
  --syntax-check` clean. Two intentional deviations from spec kept:
  (1) idempotent re-pause guard so 15a re-runs don't clobber 15b's
  unpause; (2) server-tls Secret mounted at `/usr/local/apisix/certs/server/`
  — filed as F05 with PEM-source decision (direct-Jinja insert) locked
  into T04.md. Silmarils-side uncommitted.
