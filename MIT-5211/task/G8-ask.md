# G8 — Sign-off ask for silmarils platform owner

> **What we need from you (the silmarils platform / LFI integration owner):**
> Confirm the 17 `lfi-id → upstream` mappings for the silmarils-qa AKS
> APISIX deployment. Once locked, these become the `org_routing` block
> in `iac/eng-infra/shared-k8s/ansible/variables-qa.yaml` (rendered into
> the in-cluster `apisix-org-routing` ConfigMap, read at request time by
> `org-router.lua`).
>
> Also: please confirm or correct the NBF/EIB CN→lfi-id pairs flagged
> in **FINDING F03** below — they look like a swap in the source
> template and we don't want to silently transcribe a bug.

---

## Context (1 minute read)

We are deploying Apache APISIX as the LFI edge gateway on `silmarils-qa`
AKS (ticket MIT-5211). It replaces no existing component — APISIX is
new in QA. It will run behind the existing silmarils Cloudflare Zero
Trust Tunnel at `silmarils-qa-apisix.takamul.cc`.

The routing contract is **identical** to the local docker stack
(`silmarils/apisix/`):

1. Bank presents `.p12` client cert (in dev/qa via
   `X-Forwarded-Client-Cert` header injected by the test harness;
   on-prem via F5).
2. APISIX `cert-validator.lua` plugin validates against the bank CA
   bundle, extracts CN, maps to `lfi-id` via the `cn_whitelist`.
3. APISIX `org-router.lua` plugin reads `lfi-id`, looks up the upstream
   from `org_routing.json` (this file — `silmarils.apisix.org_routing`).
4. APISIX proxies to that upstream, with `forward-auth` to `dc-tang`
   for inbound auth (separate concern).

For QA, **most upstreams are expected to be in-cluster simulators**
(`jisr-simulator.silmarils-qa.svc.cluster.local:8080`,
`mbridge-mock.silmarils-qa.svc.cluster.local:8080` — confirm exact
service names against `silmarils-qa` namespace). The exception would be
any bank whose real-LFI sandbox is reachable from `silmarils-qa` and
where you want QA traffic to go to the real endpoint instead of a sim.

---

## Half A — CN → lfi-id mapping (extracted from `apisix/apisix.yaml.template`)

These are the **17 entries currently in source-of-truth**. Please
confirm or annotate.

| CN (client cert) | lfi-id | Notes |
|---|---|---|
| `system@fab.com` | `lfi-FAB` | First Abu Dhabi Bank |
| `system@adcb.com` | `lfi-ADCB` | Abu Dhabi Commercial Bank |
| `system@adib.com` | `lfi-ADIB` | Abu Dhabi Islamic Bank |
| `system@enbd.com` | `lfi-ENBD` | Emirates NBD |
| `system@rakbank.com` | `lfi-RAK` | RAKBank |
| `system@mashreq.com` | `lfi-MASHREQ` | Mashreq |
| `system@bankofchina.com` | `lfi-BOC` | Bank of China |
| `system@dxb.icbc.com.cn` | `lfi-ICBC` | ICBC Dubai |
| `system@dib.ae` | `lfi-DIB` | Dubai Islamic Bank |
| `system@cbd.ae` | `lfi-CBD` | Commercial Bank of Dubai |
| `system@hsbc.com` | `lfi-HSBC` | HSBC |
| `system@citi.com` | `lfi-CITI` | Citi |
| `system@standardchartered.com` | `lfi-SCB` | Standard Chartered |
| `system@sjb.ae` | `lfi-SIB` | Sharjah Islamic Bank |
| **`system@nbf.ae`** | **`lfi-EIB`** | **⚠ F03: nbf.ae = National Bank of Fujairah, but maps to lfi-EIB (Emirates Islamic Bank)?** |
| **`system@mbf.ae`** | **`lfi-NBF`** | **⚠ F03: no obvious `mbf.ae` bank — looks paired with the row above** |
| `system@centralbank.com` | `lfi-CB` | Central Bank UAE |

> **F03 question (please answer):** Are the highlighted rows
> intentional aliases (e.g. testing alt-CNs against EIB/NBF chains), or
> a copy/paste swap that should be fixed in
> `apisix/apisix.yaml.template` before MIT-5211 transcribes into
> `variables-qa.yaml`? Also: should there be an `emiratesislamic` /
> `nbf.ae` CN with a matching lfi-id, given both `.p12` files exist
> under `apisix/client-certs/`?

---

## Half B — lfi-id → upstream (please fill)

For each `lfi-id`, give us the **QA** upstream. Default hypothesis is
"in-cluster simulator at `jisr-simulator.silmarils-qa.svc.cluster.local:8080`
with `path_prefix: /mbridge/ws`" but confirm per-LFI.

| lfi-id | host | port | scheme | path_prefix |
|---|---|---|---|---|
| `lfi-FAB` | | | | |
| `lfi-ADCB` | | | | |
| `lfi-ADIB` | | | | |
| `lfi-ENBD` | | | | |
| `lfi-RAK` | | | | |
| `lfi-MASHREQ` | | | | |
| `lfi-BOC` | | | | |
| `lfi-ICBC` | | | | |
| `lfi-DIB` | | | | |
| `lfi-CBD` | | | | |
| `lfi-HSBC` | | | | |
| `lfi-CITI` | | | | |
| `lfi-SCB` | | | | |
| `lfi-SIB` | | | | |
| `lfi-EIB` | | | | |
| `lfi-NBF` | | | | |
| `lfi-CB` | | | | |

(If "use the in-cluster simulator default for all", we can shortcut
this with one line and revisit per-LFI later.)

---

## Half C — confirm `dc_tang` + `forward_auth` for QA

These are the static upstreams (not per-LFI). Defaults assumed in
PLAN.md:

```yaml
dc_tang:
  host: dc-tang.silmarils-qa.svc.cluster.local
  port: 28888
  scheme: https

forward_auth:
  uri: https://dc-tang.silmarils-qa.svc.cluster.local:28888/auth/inbound
  ssl_verify: true
```

Confirm `28888` and the service name `dc-tang` are correct for the QA
namespace.

---

## Where the answers go

Paste the filled table back here, or commit it directly to this file
in `ai-coding-tickets/MIT-5211/task/G8-ask.md`. Once locked, the
values feed `variables-qa.yaml` in T06.

**Blocker status**: T06 (wiring) is the only task blocked on G8. T02,
T03, T04, T05, T07 are all unblocked and either DONE or actively
authoring.
