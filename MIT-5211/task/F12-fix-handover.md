# F12 — dc-tang ESAPI runtime failure: handover to silmarils backend team

> **Audience**: silmarils backend / controller-gateway maintainers
> (especially Pramod / Vedesh, authors of PR #163).
> **Discovered during**: MIT-5211 APISIX-on-QA verification, 2026-05-19.
> **Not a gateway issue** — APISIX, Cloudflare Tunnel, cert-validator,
> and dc-tang auth all work correctly; the failure is the very first
> instruction in the application's workflow pipeline.

## TL;DR

Every authenticated ISO API request (issuance, redemption, transfer)
fails dc-tang's workflow at **step 1** (`CheckIssuanceRequestWorkflowStep`)
with:

```
StsCd: RJCT
Desc:  java.lang.reflect.InvocationTargetException SecurityConfiguration class
       (org.owasp.esapi.reference.DefaultSecurityConfiguration) CTOR threw exception.
Caused by: ESAPI.properties could not be loaded by any means. Fail.
Caused by: Failed to load ESAPI.properties as a classloader resource.
```

The pipeline never persists an event, never publishes to Kafka, and
never reaches the mBridge / Jisr connectors. HTTP 200 is returned only
because the SOAP envelope wraps the exception as a `RJCT` receipt — no
2026-flavour real issuance has succeeded since SIL-422 deployed.

## Where it lives

```
backend/controller-gateway/platform/domain-model/
  src/main/kotlin/com/r3/dc/tanx/domainmodel/logging/WorkflowLoggingHelper.kt:42
```

```kotlin
import org.owasp.esapi.ESAPI
…
/**
 * Encodes [value] using OWASP Encoder ESAPI.encoder().encodeForHTML(), which is recognised by Checkmarx
 * as a sanitizer to break the CWE-79 taint chain. Also prevents log injection (CWE-117)
 * via MDC output. Any null input is treated as empty string.
 */
internal fun sanitize(value: String?): String = ESAPI.encoder().encodeForHTML(value ?: "")
```

This `sanitize()` is called by every MDC-context builder
(`buildStateMdcContext`, `extractPayloadFields`, `buildTransportMdcContext`),
so the first MDC write any workflow step performs trips the failure.

## Why it fails at runtime

`ESAPI.encoder()` is the public entry point of OWASP ESAPI's static
factory. It lazy-initialises a singleton `DefaultSecurityConfiguration`,
which **requires `ESAPI.properties`** to be loaded from one of (in
order of precedence):

1. The system property `org.owasp.esapi.resources` directory
2. The JVM's user.home `/.esapi/` directory
3. The user.dir `/.esapi/` directory
4. The classpath root (i.e., `src/main/resources/ESAPI.properties`)

In the current dc-tang fat jar:

| Location | Has `ESAPI.properties`? |
|---|---|
| `BOOT-INF/classes/` (silmarils' own resources) | ❌ |
| `BOOT-INF/lib/esapi-2.7.0.0.jar` (the lib itself — sometimes ships a default) | ❌ (jar verified: only `ESAPI-properties.xsd` is bundled, the actual `.properties` is not) |
| Any sibling resources jar in `BOOT-INF/lib/` | ❌ |
| The image filesystem (`find / -name ESAPI.properties`) | ❌ |
| `JAVA_TOOL_OPTIONS` / `JAVA_OPTS` / `runApp.sh` setting `-Dorg.owasp.esapi.resources=…` | ❌ |

So `loadConfigurationFromClasspath()` fails → `ConfigurationException` →
`InvocationTargetException` bubbles up through `ESAPI.encoder()` →
`sanitize()` → first call site (`buildStateMdcContext` MDC writes).

## How it got there

| Date | Commit | What |
|---|---|---|
| 2026-05-14 | `97a9f3f5` (SIL-125, "Fix for new SAST Checkmarx and Sonar security issues") | Added the `sanitize()` function — implemented as a simple regex: `value?.replace(Regex("[\r\n]"), "") ?: ""`. Wrapped MDC writes. **No ESAPI involved.** |
| 2026-05-18 | `15e7e89c` (SIL-422, "issuance checkmarkx fixes", PR #163) | Replaced the regex with `ESAPI.encoder().encodeForHTML(value ?: "")` — comment says "recognised by Checkmarx as a sanitizer to break the CWE-79 taint chain". Did **not** add an explicit `org.owasp:esapi` dependency in `domain-model/build.gradle`, did **not** add `ESAPI.properties` to resources, did **not** set the system property in `runApp.sh`. |
| 2026-05-18 14:10 UTC | ACR build | Image `silmarilsacr.azurecr.io/silmarils:release-v-128-20260518-175255` published — current QA-deployed dc-tang. |
| 2026-05-19 | This handover | Discovered during MIT-5211 ISO API smoke. |

## Reproduction (no APISIX involved)

The bug fires for any LFI-authenticated ISO API call. Easiest test —
hit dc-tang directly from inside the cluster, bypassing APISIX:

```bash
kubectl -n silmarils-qa exec deploy/dc-tang -- sh -c "
  curl -ksS -o /tmp/r -w 'HTTP %{http_code}\n' \
    -X POST https://localhost:28888/iso/api/v1/issuance-platform \
    -H 'Content-Type: text/xml' \
    -H 'X-LFI-ID: lfi-ADCB' \
    -H 'X-LFI-API-KEY: <any-valid-active-ADCB-key-from-gateway-DB>' \
    --data-binary @/tmp/probe.xml ; tail -c 400 /tmp/r"
```

Even a stripped-down empty pacs.009 with the Document wrapper triggers
the same failure — confirming the call path is auth → MDC → sanitize →
ESAPI → throw, regardless of payload content.

dc-tang log signature for every failure:
```
ERROR c.r.d.t.s.t.h.SyncRouteDispatcherImpl :
  Workflow 'CheckIssuanceRequestWorkflowStep' failed —
  correlationId: <uuid>:
  java.lang.reflect.InvocationTargetException SecurityConfiguration class …
   at org.owasp.esapi.ESAPI.encoder(ESAPI.java:104)
   at com.takamul.silmarils.isoapi.transaction.handler.issuance
        .IsoIssuanceRequestHandler.executeRequest(IsoIssuanceRequestHandler.kt:87)
```

## Three fix options (recommendation in bold)

### ✅ **Fix 1 (recommended): Replace ESAPI with OWASP Java Encoder**

OWASP **Java Encoder** is a small, zero-config, modern alternative to
ESAPI that exposes the same `encodeForHTML(...)` semantics and is
recognised by Checkmarx as a CWE-79 / CWE-117 sanitiser.

```kotlin
// before
import org.owasp.esapi.ESAPI
internal fun sanitize(value: String?): String = ESAPI.encoder().encodeForHTML(value ?: "")

// after
import org.owasp.encoder.Encode
internal fun sanitize(value: String?): String = Encode.forHtml(value ?: "")
```

In `backend/controller-gateway/platform/domain-model/build.gradle`:

```groovy
dependencies {
    implementation "com.fasterxml.jackson.module:jackson-module-kotlin:$jacksonModuleKotlinVersion"
    implementation "org.owasp.encoder:encoder:1.3.1"   // ← add
    …
}
```

…and ideally remove the inherited `esapi` runtime dep:

```groovy
configurations.all {
    exclude group: 'org.owasp.esapi', module: 'esapi'
}
```

**Pros**: no config files, no system properties, no classpath dance,
much smaller artefact, same Checkmarx recognition.
**Cons**: none material.
**Effort**: 1-line code change + 1-line build.gradle change + rebuild.

### Fix 2 — Revert to the SIL-125 regex sanitiser

This is what the codebase had between SIL-125 (May 14) and SIL-422
(May 18) and works at runtime. The reason SIL-422 moved away from it
was Checkmarx; if the team is willing to add a `// nosem`-style
suppression or annotate it for Checkmarx, this is fine:

```kotlin
internal fun sanitize(value: String?): String =
    value?.replace(Regex("[\r\n]"), "") ?: ""
```

**Pros**: zero dependencies, simplest possible code.
**Cons**: Checkmarx will likely re-flag CWE-117 / CWE-79 unless
suppressed; suppressions need formal sign-off.
**Effort**: 1-line code change + a SAST suppression entry.

### Fix 3 — Keep ESAPI but ship its config

Add the missing config files so ESAPI can initialise:

1. Add `backend/controller-gateway/platform/domain-model/src/main/resources/ESAPI.properties`
   (minimal config — copy the
   [reference defaults](https://github.com/ESAPI/esapi-java-legacy/blob/develop/configuration/esapi/ESAPI.properties)
   and trim to what you actually use; for log/HTML encoding you can
   keep most of it as upstream).
2. Add `backend/controller-gateway/platform/domain-model/src/main/resources/validation.properties`
   (ESAPI loads this alongside; even an empty file with the standard
   structure satisfies the bootstrap).
3. Optionally pin `org.owasp.esapi:esapi:2.7.0.0` explicitly in
   `domain-model/build.gradle` so the version is intentional rather
   than transitive:
   ```groovy
   implementation "org.owasp.esapi:esapi:2.7.0.0"
   ```

**Pros**: keeps the Checkmarx-recognised ESAPI brand.
**Cons**: ships two non-trivial config files; ESAPI's init has known
edge cases on newer JVMs (its bootstrap reflectively touches a lot);
the silmarils backend currently uses ESAPI for one (1) line of code,
which is a high cost for that single call site.
**Effort**: 2 new resource files + classpath verification.

## Verification once a fix is rolled out

Run the MIT-5211 happy-path curl (§2 Level 3 of
`MIT-5211/task/iso-api-tests.md`) — same cert, same key, same payload
— and confirm:

1. Response: `HTTP 200` (unchanged).
2. Body: `<StsCd>ACCP</StsCd>` (was `RJCT` with the ESAPI Desc).
3. Gateway DB: at least one row in `silmarils_qa_gateway.events`
   keyed by the correlationId echoed in the response.
4. dc-tang logs: `Step [CheckIssuanceRequestWorkflowStep] COMPLETED`
   followed by subsequent step entries; no
   `InvocationTargetException` / `ESAPI`.

```bash
# After fix:
MSG_ID="<from your curl response OrgnlMsgId>"
kubectl -n silmarils-qa logs deploy/dc-tang --since=5m | grep -i "$MSG_ID"
# should NOT contain 'ESAPI' or 'InvocationTargetException' anywhere.

kubectl -n data-silmarils exec postgres-0 -- bash -c \
  "PGPASSWORD=\$POSTGRES_PASSWORD psql -U postgres -d silmarils_qa_gateway -c \"
   SELECT timestamp, event_type FROM events
    WHERE details::text ILIKE '%${MSG_ID}%' ORDER BY timestamp;\""
# should return >= 1 row.
```

## Out of scope for this handover (won't fix as part of F12)

- MIT-5211 / APISIX gateway side — already proven green;
  HTTP 200 + auth + cert + parse + routing all work today.
- Controller→gateway `lfi_api_config` sync (F11) — separate ticket at
  `ai-coding-tickets/MIT-NEXT-controller-gateway-key-sync/`.
- F03 CN→lfi-id swap for NBF/EIB — separate platform-owner question.

## References

- The bug in source: silmarils PR
  [#163 SIL-422 issuance checkmarkx fixes](https://github.com/takamulai/silmarils/pull/163),
  file `backend/controller-gateway/platform/domain-model/src/main/kotlin/com/r3/dc/tanx/domainmodel/logging/WorkflowLoggingHelper.kt`,
  line 42.
- ESAPI runtime trace: this dc-tang stack frame is the smoking gun:
  ```
  at org.owasp.esapi.ESAPI.encoder(ESAPI.java:104)
  at com.takamul.silmarils.isoapi.transaction.handler.issuance
       .IsoIssuanceRequestHandler.executeRequest(IsoIssuanceRequestHandler.kt:87)
  ```
- Recommended replacement library:
  [OWASP Java Encoder](https://owasp.org/www-project-java-encoder/).
- Sample verified responses (200 + RJCT + ESAPI body) captured in
  `MIT-5211/task/verified/T08-tunnel-pacs009-isue-adcb*.xml`.
