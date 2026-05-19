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

## F10 — Liveness probe `HTTP /healthz` requires a route that doesn't exist

**What**: T03's `apisix-deployment.yaml.j2` configured `livenessProbe:
httpGet /healthz on port 19888`. PLAN.md §Production hardening said
this would be "a `public-api`-served `/healthz` route in `apisix.yaml`
for HTTP liveness". That route was never added to
`apisix.yaml.template` or to `apisix-routes.yaml.j2`. Result: pods
respond Ready (TCP socket on 19880 works) then liveness probe fails
within 15s → kubelet restarts the pod → CrashLoopBackOff-like
flapping despite the process running.

**Why it matters**: First-bring-up pods restarted exactly once each
on liveness deadline before stabilising via a deployment patch.
Without the patch, restart count would climb indefinitely (1 every
~45s) and PDB+HPA behaviour would degrade.

**How we handle it** (applied post-T08): patched deployment to use
`tcpSocket :19888` for liveness (gateway port — already bound when
APISIX is healthy). Same probe shape as readiness/startup. Eliminates
the dependency on a routes-CM-supplied healthz route.

**Do NOT**: Add a `public-api`-backed `/healthz` route without first
deciding whether it should bypass cert-validator (currently every
route inherits `plugin_config_id: 1` which requires X-Forwarded-Client-Cert).
A healthz route that requires a bank cert defeats the purpose of
kubelet probing. If we want HTTP liveness, the route needs a separate
plugin_config without cert-validator.

---

## F09 — `readOnlyRootFilesystem: true` requires emptyDirs for every nginx temp dir

**What**: T03's deployment template set `readOnlyRootFilesystem: true`
per PLAN.md §Production hardening. APISIX standalone mode writes to
several paths at every startup:
- `/usr/local/apisix/conf/nginx.conf` (regenerated by `apisix init`)
- `/usr/local/apisix/client_body_temp` (nginx client body temp)
- `/usr/local/apisix/proxy_temp`, `fastcgi_temp`, `uwsgi_temp`, `scgi_temp`
- `/tmp/disk_cache_one` (proxy disk cache configured by nginx.conf)

Each one needs a writable emptyDir mount. Partial fix (just `/conf`
and `/tmp`) bought one step's worth of progress before the next
nginx temp dir tripped.

**How we handle it** (applied post-T08): `readOnlyRootFilesystem`
**disabled** on the main container in the template (commented out
with F09 reference). PSA Restricted does NOT require it
(`runAsNonRoot`, `seccompProfile`, `cap drop ALL`,
`allowPrivilegeEscalation: false` remain enforced). Re-enable in a
follow-up by adding emptyDir mounts for every nginx temp dir AND
ensuring `apisix init` can rewrite nginx.conf.

**Do NOT**: Re-enable readOnlyRootFilesystem without first auditing
the full list of write targets via a strace/audit, OR by switching
to a different startup path that doesn't regenerate nginx.conf.

---

## F08 — APISIX needs writable `/usr/local/apisix/conf` (initContainer pattern)

**What**: APISIX `init` regenerates `/usr/local/apisix/conf/nginx.conf`
on every startup and reads stock files like `config-default.yaml` and
`mime.types` from the same directory. With subPath ConfigMap mounts
for `apisix.yaml` / `config.yaml` / `org_routing.json`, the rest of
`/conf` is the image's read-only stock content. Without further
volume choreography, `apisix init` can't write nginx.conf there.

**How we handle it** (applied post-T08): emptyDir mounted at
`/usr/local/apisix/conf`, seeded by an initContainer that runs
`cp -rn /usr/local/apisix/conf/. /scratch/` (the same image as the
main container, so stock files are available to copy). subPath CM
mounts in the main container land specific files into this emptyDir
overlay. **`cp -an` does NOT work** — the `-a` flag tries to preserve
timestamps on the emptyDir mountpoint, which fails for non-root user
636 with "Operation not permitted". Use `cp -rn` (recursive, no
preserve) instead.

**Do NOT**: Skip the initContainer and just mount emptyDir at `/conf`
— APISIX `init` will fail looking for `config-default.yaml` etc.
Don't try to bake config-default.yaml into the silmarils image build
either; it's image-version-coupled and would need re-baking on every
upstream upgrade.

---

## F12 — dc-tang OWASP ESAPI init exception when processing pacs.009 ISUE

**What**: Once the inbound API key validation passes (HTTP 200), dc-tang
proceeds to process the pacs.009 body and immediately fails with:
```
StsCd: RJCT
Desc: java.lang.reflect.InvocationTargetException
      SecurityConfiguration class
      (org.owasp.esapi.reference.DefaultSecurityConfiguration)
      CTOR threw exception.
```
Confirmed against `silmarils-qa` on 2026-05-18 with a valid plaintext
key (ADCB) — the ESAPI CTOR fails before any business validation runs.

**Why it matters**: Outside MIT-5211's scope (APISIX is fully proven
— request reached dc-tang, was authenticated, and was parsed as a
valid pacs.009.001.12). But it's the immediate next blocker for any
real ISO API flow through silmarils-qa. The Postman test asserts
`StsCd = ACCP`; that assertion will fail until ESAPI initialises
cleanly.

**How we handle it**: Hand off to silmarils application team — this
is a dc-tang runtime/classpath issue (ESAPI config file missing or
unreadable, or jvm `-Dorg.owasp.esapi.resources` system property
pointing at the wrong dir). Out of MIT-5211 scope.

**Do NOT**: Block MIT-5211 verification on this. The HTTP 200 +
`OrgnlMsgId echo` proves the gateway-side contract.

---

## F11 — Controller→gateway `lfi_api_config` sync stopped propagating

**What**: dc-tanc's `POST /api/v1/organizations/<orgId>/api-config/MBRIDGE_JISR/inbound/generate`
writes the new key row to the **controller DB**
(`silmarils_qa_controller`). dc-tang reads from a **separate gateway
DB** (`silmarils_qa_gateway`). The two DBs sync via Kafka events (per
the playbook env vars). On 2026-05-18 in silmarils-qa, controller-side
INSERTs from ~06:00 UTC onward stopped propagating to the gateway DB
(gateway's newest ADCB row was 05:47 UTC; my key generated at 10:42
UTC never appeared on the gateway side). Earlier-the-same-day INSERTs
DID propagate, so this is a regression, not a config gap.

**Why it matters**: Newly-generated API keys cannot authenticate any
real LFI request until sync resumes. Affects every LFI in
silmarils-qa, not just ADCB. Pre-existing silmarils platform issue;
unrelated to MIT-5211 / APISIX.

**How we handle it**: Filed as a separate ticket placeholder under
`ai-coding-tickets/MIT-NEXT-controller-gateway-key-sync/`. For
MIT-5211 verification only, applied a one-off `UPDATE` on the gateway
DB to replace the hash on ADCB's PRIMARY-ACTIVE row with the new
key's hash (preserves the row identity, sidesteps the
`uq_lfi_api_config_active` unique constraint, and avoids needing the
event sync). Restarted dc-tang to drop any in-memory cache. With both
applied, HTTP 200 proven (`task/verified/T08-tunnel-pacs009-isue-adcb-200.xml`).

**Do NOT**: Use the SQL UPDATE workaround for any real flow. It
bypasses business logic, audit, and event semantics. Revoke the
temporary ADCB key (`lfi_MOt6n-C9…`) once the sync is fixed and a
freshly-generated key propagates cleanly.

**Update (2026-05-19)**: A user-supplied key
`lfi_SJ8jhxqyCRwVaUJ9AuoUBBxkE8Ihs7vOnQvmDodmJvA` also authenticates
as ADCB and returns HTTP 200 (artefact:
`task/verified/T08-tunnel-pacs009-isue-adcb-userkey-200.xml`). Its
hash must match one of the pre-existing ACTIVE rows in the gateway
DB (likely a 2026-05-14 SECONDARY entry). This **does not change the
F11 root cause** — the controller→gateway sync for *new* generations
is still broken. It just means there's a known-good key we can use
for ongoing smoke testing without depending on the SQL workaround.
The workaround (UPDATE on PRIMARY-ACTIVE) and the orphan controller
key are **intentionally left in place** pending F11 investigation
(per user direction 2026-05-19); will be reverted as part of the
follow-up ticket's cleanup definition-of-done.

---

## F07 — First-bring-up `image.tag` has no real ACR tag to point at yet

**What**: T02's `apisix-image.yaml` workflow emits
`APISIX_VERSION="$(git describe --tags --always)"`. Until that workflow
runs on `main` for the first time **and** the silmarils repo has at
least one git tag reachable from HEAD, `git describe --tags --always`
returns just the short SHA (`g<sha>`), and no image with that tag
exists in ACR until the workflow run completes. T06 set
`silmarils.apisix.image.tag: "main"` as a placeholder, which **will
not resolve in ACR** — `:main` is not pushed by the workflow.

**Why it matters**: Running T08 (`ansible-playbook ... --tags
apisix-base,apisix-silmarils-lfi`) before T02's CI workflow has
completed → `ImagePullBackOff` on the apisix Deployment → pods never
go Ready → T08 fails. This is also true if T02 CI has run but
`variables-qa.yaml` was not updated to the real tag CI emitted.

**How we handle it (two paths, pick one before T08)**:

1. **Manual tag bump (recommended for first apply)**: After T02's
   silmarils PR merges and CI runs, read the tag from
   `az acr repository show-tags --name silmarilsacr --repository silmarils-apisix`,
   update `variables-qa.yaml`'s `silmarils.apisix.image.tag` to that
   exact value, commit, then run T08. One extra step but no workflow
   changes — keeps tags immutable.
2. **Add moving `:main` tag to the workflow**: Amend T02's
   `apisix-image.yaml` `Compute image tags` step to also push `:main`
   (similar to how it currently pushes `:latest` only on tag refs).
   T08 then works on first run with `image.tag: "main"`. Trades tag
   immutability for ergonomics — acceptable for first-bring-up only;
   revert before SIT/UAT/PROD.

**Do NOT**: Use `imagePullPolicy: Always` to mask this. The Deployment
template already conditions pullPolicy on the tag string
(`Always` only when tag == `latest`). Forcing `Always` everywhere
hides bad tag values and slows pod start.

---

## F06 — Bank `.p12` passphrase is `"password"`, not empty

**What**: T04's spec assumed the 17 bank `.p12` files use an empty
passphrase. They do not — passphrase is **`"password"`**, matching
`silmarils/apisix/setup-certs.sh:7` (`PASSWORD="password"` — same
convention used across silmarils for `genericKeystore.p12`,
`truststore.p12`, `mbridge-connector-keystore.p12`, etc.).
Verified via direct CLI test: `openssl pkcs12 -passin pass:` fails
with `Mac verify error`; `openssl pkcs12 -passin pass:password`
succeeds and extracts the CA chain cleanly.

**Why it matters**: 15b's CA-bundle assembly step (`openssl pkcs12 -in
... -cacerts -nokeys -nomacver -passin pass:...`) needs the right
passphrase or the playbook fails on every run. Also affects the
silmarils convention going forward — any new bank cert must use the
same password or `variables-qa.yaml` needs a per-cert override
mechanism (out of scope for now).

**How we handle it**: 15b reads the passphrase from a new variable
**`silmarils.apisix.client_certs.passphrase`** (default `"password"`).
T06 must add this to the `silmarils.apisix` block of
`variables-qa.yaml` (explicit value, not relying on the default —
makes rotation obvious).

**Do NOT**: Hardcode `"password"` in the playbook. Even though it's
the only value today, future cert reissuance might force a change and
a hardcoded literal would silently work in dev/qa but break on the
first SIT/UAT/PROD rotation.

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
