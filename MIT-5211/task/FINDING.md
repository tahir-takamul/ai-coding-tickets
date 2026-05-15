# FINDING — MIT-5211

> Numbered gotchas discovered during execution. Each entry follows:
> **What / Why it matters / How we handle it / Do NOT**.

---

## F01 — `cloudflared` Deployment has no dedicated ServiceAccount

**What**: The `cloudflared` Deployment in
`silmarils/iac/eng-infra/shared-k8s/ansible/playbooks/05-cloudflare-tunnel.yaml`
has **no `spec.template.spec.serviceAccountName`** set. It runs in
namespace `{{ namespaces.ingress_nginx }}` → `"ingress-nginx"` (resolved
from `variables-qa.yaml:26`) with pod label `app: cloudflared`. It
therefore inherits the namespace `default` SA.

**Why it matters**: PLAN.md §Production hardening and `T05.md` both
proposed a NetworkPolicy ingress rule selecting `cloudflared` by
ServiceAccount. That selector won't work — there is no
`cloudflared`-specific SA to match. The default SA is shared by every
pod in `ingress-nginx`.

**How we handle it**: T05's `apisix-networkpolicy.yaml.j2` selects
cloudflared traffic by `namespaceSelector` (`kubernetes.io/metadata.name:
ingress-nginx`) **combined with** `podSelector` (`matchLabels: { app:
cloudflared }`). Both selectors required — `namespaceSelector` alone is
too broad (lets all of `ingress-nginx` in), `podSelector` alone is too
narrow (depends on the apisix-silmarils namespace not also having an
`app=cloudflared` pod, which is fragile).

**Do NOT**: Loosen this to a pure namespaceSelector. The `ingress-nginx`
namespace also runs the regular nginx-ingress controller; we do not want
that to be able to reach APISIX directly bypassing the tunnel-only
contract. Equally, do not switch to ServiceAccount-based selection
without first patching the cloudflared Deployment to use a dedicated SA
(out of scope for this ticket).

---

## F02 — `kubernetes.yaml` "placeholder" is a 3-line comment block, not a code stub

**What**: PLAN.md §Files to create / edit says
> `iac/eng-infra/shared-k8s/ansible/kubernetes.yaml      # replace placeholder lines 199-200`

The actual content at lines 198-200 is just a comment banner:
```yaml
        #############################
        # 15 — Placeholder (removed)
        #############################
```
There is no live task block to replace — the section is intentionally
empty pending this work.

**Why it matters**: The T06 instruction "replace the placeholder" is a
slight misnomer. We are **inserting** two include_tasks blocks into a
deliberately reserved gap, not editing/replacing logic.

**How we handle it**: T06 keeps the `# 15 — ...` banner (rename it to
`# 15a-b — APISIX`) and adds the two `include_tasks` calls inside the
gap between this banner and the `# 16-17 — Monitoring` banner. Same
visual style as surrounding sections.

**Do NOT**: Delete the comment block — it preserves the section
numbering used by the other playbooks.

---

## F03 — Apparent CN→lfi-id swap in `apisix.yaml.template` for NBF/EIB

**What**: In `silmarils/apisix/apisix.yaml.template` lines 45-48, the
`cn_whitelist` entries map:
```yaml
"system@nbf.ae":  lfi-EIB        # nbf.ae = National Bank of Fujairah, but → lfi-EIB
"system@mbf.ae":  lfi-NBF        # mbf.ae = ??? (no obvious bank), but → lfi-NBF
```
There is no `system@emiratesislamic.*` entry, and `emiratesislamicbank.p12`
exists in `client-certs/`. Looks like a copy/paste swap, or these are
test placeholder values. There is also no CN entry that maps to a
plausible `mbf.ae` bank.

**Why it matters**: T06 transcribes these 17 entries verbatim into
`variables-qa.yaml`. If F03 is a typo in the source-of-truth template,
QA traffic from EIB or NBF banks will get the wrong `X-LFI-ID` and
their `org-router` lookups will go to the wrong upstream.

**How we handle it**: Block T06 until silmarils owners confirm whether
F03 is intentional (testing aliases) or a bug. If a bug, fix in the
**source-of-truth template** (`apisix/apisix.yaml.template`) first via
a separate PR, then transcribe into `variables-qa.yaml`. Do not
silently "correct" the values in `variables-qa.yaml` only — that
creates a divergence between local dev and QA.

**Do NOT**: Just retype what looks "more correct" without confirmation.
Local-dev test users may rely on the current mapping for known reasons
(certs were generated against these CNs).

---

## F05 — server-tls Secret mounted at `/usr/local/apisix/certs/server` — T04 must align

**What**: T03's `apisix-deployment.yaml.j2` mounts the `apisix-server-tls`
Secret as a directory at `/usr/local/apisix/certs/server` (lines 118-120).
cert-manager Secrets carry the keys `tls.crt`, `tls.key`, `ca.crt`. So the
in-pod paths are:
- `/usr/local/apisix/certs/server/tls.crt`
- `/usr/local/apisix/certs/server/tls.key`
- `/usr/local/apisix/certs/server/ca.crt` (unused — silmarils-ca-issuer
  cert chain is internal-only; APISIX presents this server cert to
  cloudflared which terminates with `origin_no_tls_verify: true`).

**Why it matters**: T04 renders the `apisix-routes` ConfigMap from
`apisix/apisix.yaml.template`, which contains an `ssls:` block with
`__SSL_CERT__` / `__SSL_KEY__` placeholders. Two ways to fill them in:
1. **Direct-Jinja insert** of PEMs read from the materialized Secret via
   `k8s_info` (PLAN.md §15b step 4 wording).
2. **Reference the mount paths** so APISIX reads PEM files at startup;
   no PEM material baked into the CM.

The two approaches are mutually compatible (you could do both), but the
operational characteristics differ: (1) requires a CM rewrite + pod
rollout on cert rotation; (2) only needs a pod restart.

**How we handle it**: For first iteration, **direct-Jinja insert** —
keeps T04 simple, the silmarils-ca-issuer cert is 90d and HPA is 2-4
pods so rotation churn is tolerable. The mount stays in T03's Deployment
template as defence-in-depth (and for the future approach 2 transition).
T04's `apisix-routes.yaml.j2` reads `apisix_server_cert_pem` /
`apisix_server_key_pem` from a prior `k8s_info` task and inserts them
into the `ssls.cert` / `ssls.key` fields.

**Do NOT**: Use a Jinja path-only reference (`cert: "/usr/local/apisix/certs/server/tls.crt"`)
without first verifying APISIX 3.11 standalone mode accepts file paths
in `ssls.cert` instead of PEM strings — older versions accepted only
inline PEM. If the path form is desired, prototype it first.

---

## F04 — `apisix.yaml.template` upstream port for dc-tang matches PLAN.md

**What**: PLAN.md assumes dc-tang upstream at port 28888 (NetworkPolicy
egress allow rule). `apisix.yaml.template` line 234 confirms:
`"dc-tang:28888": 1` on `scheme: https`. The forward-auth URI in routes
5-8 also uses `https://dc-tang:28888/auth/inbound`.

**Why it matters**: Not a deviation — just confirming the plan number
against source-of-truth. T05's NetworkPolicy egress rule of port
28888/TCP is correct.

**How we handle it**: No action; entry exists as a positive
cross-reference for future contributors who might wonder where 28888
came from.

**Do NOT**: Hardcode 28888 in templates — pull it from
`silmarils.apisix.dc_tang.port` so the variable schema is the single
source.
