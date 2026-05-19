# ISO API smoke test — silmarils-qa APISIX gateway

> **Audience**: QA testers / anyone with a laptop on the internet.
> **Goal**: prove that a bank-flavoured pacs.009 ISUE request travels
> end-to-end through the new APISIX LFI edge gateway at
> `silmarils-qa-apisix.takamul.cc`.
>
> No VPN, no kubeconfig, no Cloudflare Access auth needed. Three
> copy-paste blocks below.

---

## 1. Prerequisites

| Need | Why |
|---|---|
| `curl` | Send the request |
| `openssl` | Extract bank certificate from `.p12` to PEM |
| `jq` | URL-encode the PEM for the `X-Forwarded-Client-Cert` header |
| `uuidgen` | Generate fresh `MsgId` / `UETR` per call |
| The silmarils repo cloned at `~/DevWorkspace/apisix/silmarils/` | Source for the bank `.p12` files (`apisix/client-certs/`) |
| A valid `X-LFI-API-KEY` for the bank you're testing as | Ask the silmarils platform team. Key plaintext is only available at generation time; older keys can't be looked up. (For ADCB on silmarils-qa right now, ask Mohd / Tahir for the current key.) |

If you're on a Mac and any of the CLI tools are missing, `brew install
openssl jq` covers the gaps; `uuidgen` and `curl` ship with macOS.

---

## 2. Three depth levels

Run them in order. Each one adds **exactly one more layer** to the
path being exercised — when one fails, you know which layer is broken.

### Level 1 — Tunnel + APISIX only (no cert, no payload)

```bash
curl -i https://silmarils-qa-apisix.takamul.cc/healthz
```

Expected:

```http
HTTP/2 200
content-type: application/json
server: cloudflare
…

{"status":"ok","service":"apisix-silmarils"}
```

What's proven if green: DNS resolves, Cloudflare Tunnel is up,
cloudflared in-cluster is up, APISIX Service is reachable, APISIX
itself is serving requests. If this fails, **don't bother with the
next two** — the foundation is broken.

---

### Level 2 — Adds cert validation + dc-tang auth (no API key)

Proves the gateway accepts a real bank cert, sets `X-LFI-ID`,
forwards to dc-tang, and dc-tang's auth filter speaks back. We
deliberately omit the API key so dc-tang rejects — the **shape** of
the rejection is what matters.

```bash
# Once per terminal: extract ADCB public cert from .p12
openssl pkcs12 \
  -in ~/DevWorkspace/apisix/silmarils/apisix/client-certs/abudhabicommercialbank.p12 \
  -clcerts -nokeys -nodes -passin pass:password 2>/dev/null \
  | openssl x509 -outform PEM > /tmp/adcb.pem

# Each test: POST with cert but no API key
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H 'Accept: text/xml' \
  -H "X-Forwarded-Client-Cert: $(jq -sRr @uri < /tmp/adcb.pem)" \
  --data-binary '<x/>'
```

Expected:

```http
HTTP/2 401
content-type: text/xml; charset=UTF-8

<SOAP-ENV:Envelope …>
  …
  <ns2:StsCd>RJCT</ns2:StsCd>
  <ns2:Desc>Missing X-LFI-API-KEY header</ns2:Desc>
  …
```

What's proven if green: APISIX `cert-validator` accepted the ADCB
cert, mapped its CN to `X-LFI-ID: lfi-ADCB`, proxy-rewrote the URI,
and dc-tang responded with a structured ISO 20022 acknowledgement.
The 401 is *the right error* — anything else (e.g. an HTML page, a
plain 403, a 5xx) means the gateway plumbing is broken somewhere.

---

### Level 3 — Full happy path with a valid API key (HTTP 200)

Run as a single block. Builds a fresh pacs.009.001.12 ISUE payload,
fills in dynamic fields, and sends it through the gateway. Replace
`<PASTE_KEY_HERE>` with the current valid `X-LFI-API-KEY` you got
from the platform team.

```bash
# (a) Write a known-good pacs.009.001.12 ISUE template — once per terminal
cat > /tmp/pacs009-isue.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
  <SOAP-ENV:Header>
    <Ver>01</Ver>
    <SndDtTm>__SND_DT__</SndDtTm>
    <MsgTp>ip.200.pacs.009</MsgTp>
    <MsgDrn>R</MsgDrn>
    <Network>MBRIDGE</Network>
  </SOAP-ENV:Header>
  <SOAP-ENV:Body>
    <Document xmlns="urn:iso:std:iso:20022:tech:xsd:pacs.009.001.12">
      <FICdtTrf>
        <GrpHdr>
          <MsgId>__MSG_ID__</MsgId>
          <CreDtTm>__SND_DT__</CreDtTm>
          <NbOfTxs>1</NbOfTxs>
          <SttlmInf><SttlmMtd>CLRG</SttlmMtd></SttlmInf>
        </GrpHdr>
        <CdtTrfTxInf>
          <PmtId>
            <InstrId>__MSG_ID__</InstrId>
            <EndToEndId>__E2E_ID__</EndToEndId>
            <TxId>__MSG_ID__</TxId>
            <UETR>__UETR__</UETR>
          </PmtId>
          <PmtTpInf><CtgyPurp><Cd>ISUE</Cd></CtgyPurp></PmtTpInf>
          <IntrBkSttlmAmt Ccy="AED">1000000.00</IntrBkSttlmAmt>
          <IntrBkSttlmDt>__STT_DT__</IntrBkSttlmDt>
          <Dbtr><FinInstnId>
            <ClrSysMmbId><MmbId>003</MmbId></ClrSysMmbId>
            <LEI>213800RWVKKIRX1AUH58</LEI>
            <Nm>Abu Dhabi Commercial Bank</Nm>
          </FinInstnId></Dbtr>
          <DbtrAcct><Id><Othr><Id>M-B-ADCBAEAA008</Id></Othr></Id></DbtrAcct>
          <Cdtr><FinInstnId>
            <ClrSysMmbId><MmbId>003</MmbId></ClrSysMmbId>
            <LEI>213800RWVKKIRX1AUH58</LEI>
            <Nm>Abu Dhabi Commercial Bank</Nm>
          </FinInstnId></Cdtr>
          <CdtrAcct><Id><Othr><Id>M-B-ADCBAEAA008</Id></Othr></Id></CdtrAcct>
          <RmtInf><Ustrd>Issuance request via ISO API</Ustrd></RmtInf>
          <SplmtryData><Envlp><Cnts>
            <ParamId>M-B-ADCBAEAA008e-AEDADCBAEA00001</ParamId>
            <Rsn>New Issuance application</Rsn>
          </Cnts></Envlp></SplmtryData>
        </CdtTrfTxInf>
      </FICdtTrf>
    </Document>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
EOF

# (b) Fill fresh dynamic fields (MsgId, UETR, timestamps) — run before each curl
sed -i.bak \
  -e "s|__MSG_ID__|$(uuidgen | tr -d '-' | head -c 35)|g" \
  -e "s|__E2E_ID__|$(uuidgen | tr -d '-' | head -c 35)|g" \
  -e "s|__UETR__|$(uuidgen | tr 'A-Z' 'a-z')|g" \
  -e "s|__SND_DT__|$(date -u +%Y-%m-%dT%H:%M:%SZ)|g" \
  -e "s|__STT_DT__|$(date -u +%Y-%m-%d)|g" \
  /tmp/pacs009-isue.xml

# (c) Run the curl. Replace <PASTE_KEY_HERE> with the real ADCB X-LFI-API-KEY.
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H 'Accept: text/xml' \
  -H "X-Forwarded-Client-Cert: $(openssl pkcs12 \
        -in ~/DevWorkspace/apisix/silmarils/apisix/client-certs/abudhabicommercialbank.p12 \
        -clcerts -nokeys -nodes -passin pass:password 2>/dev/null \
        | openssl x509 -outform PEM \
        | jq -sRr @uri)" \
  -H "X-LFI-API-KEY: <PASTE_KEY_HERE>" \
  --data-binary @/tmp/pacs009-isue.xml
```

Expected:

```http
HTTP/2 200
content-type: text/xml; charset=UTF-8

<SOAP-ENV:Envelope …>
  …
  <ns2:RctDtls>
    <ns2:ReqHdlg>
      <ns2:OrgnlMsgId>{your generated MsgId}</ns2:OrgnlMsgId>
      <ns2:OrgnlMsgNmId>pacs.009.001.12</ns2:OrgnlMsgNmId>
      <ns2:StsCd>{ACCP or RJCT}</ns2:StsCd>
      <ns2:Desc>…</ns2:Desc>
    </ns2:ReqHdlg>
  </ns2:RctDtls>
  …
```

**Two acceptable outcomes** — both prove the gateway path is healthy:

1. **`StsCd: ACCP`** — full happy path. dc-tang accepted the
   issuance for processing.
2. **`StsCd: RJCT`** with `Desc: …InvocationTargetException…ESAPI…` —
   **dc-tang application bug** (OWASP ESAPI initialisation), tracked
   as finding F12 in `FINDING.md`. **NOT a gateway issue** — the
   HTTP 200 + `OrgnlMsgId echoed` + `OrgnlMsgNmId=pacs.009.001.12`
   prove auth, parsing, and routing all succeeded. Hand this case
   to the silmarils application team.

---

## 3. Test matrix — happy / sad / edge with verified expected responses

All 11 cases below were **executed live** against
`silmarils-qa-apisix.takamul.cc` on 2026-05-19 and the responses
recorded verbatim. Use them as a regression baseline.

### Summary

| ID | Category | What it sends | HTTP | Body shape | Rejected by | Verdict if seen |
|---|---|---|---|---|---|---|
| **H1** | happy | `GET /healthz` (no cert, no payload) | **200** | `{"status":"ok",…}` | — (responded by APISIX `mocking` plugin) | Gateway is reachable end-to-end. |
| **H2** | happy | pacs.009 ISUE with valid ADCB cert + valid `X-LFI-API-KEY` | **200** | `admi.098.001.01` receipt, `OrgnlMsgId` echoed, `StsCd=RJCT` with ESAPI Desc (F12) | — (dc-tang processed) | Gateway path fully proven. RJCT body is dc-tang F12 (separate bug). |
| **S1** | sad | POST issuance with **no cert** | **403** | `{"error":"client certificate required"}` | APISIX `cert-validator` | Cert-required check is enforced. |
| **S2** | sad | POST issuance with **self-signed cert** (unknown CN) | **403** | `{"error":"certificate validation failed"}` | APISIX `cert-validator` | CN-whitelist check is enforced. |
| **S3** | sad | POST issuance with valid cert but **no API key** | **401** | `admi.098.001.01`, `StsCd=RJCT`, `Desc=Missing X-LFI-API-KEY header` | dc-tang `LfiApiAuthFilter` | Cert passed, dc-tang spoke. API-key check is enforced. |
| **S4** | sad | POST issuance with valid cert but **bogus API key** | **401** | `admi.098.001.01`, `StsCd=RJCT`, `Desc=Authentication failed` | dc-tang `LfiApiAuthFilter` | Cert passed, dc-tang spoke. Key-hash mismatch correctly rejected. |
| **S5** | sad | POST issuance with valid cert + valid key + **stale repo payload** (`silmarils/apisix/qa-tests/issuance-request-soap.xml`, lacks `<Document>` wrapper) | **404** | empty (Spring-Security headers) | dc-tang Spring-WS `EndpointNotFound` | dc-tang received the request but can't route a payload missing the pacs.009.001.12 namespace wrapper. (Stale-repo-file finding.) |
| **E1** | edge | `GET` on the POST-only `/api/v1/issuance-platform` | **404** | `{"error_msg":"404 Route Not Found"}` | APISIX route matcher | Method-restricted route enforces method. |
| **E2** | edge | POST to a totally unknown path | **404** | `{"error_msg":"404 Route Not Found"}` | APISIX route matcher | No unintended catch-all. |
| **E3** | edge | `POST` to `/healthz` (which is GET-only) | **404** | `{"error_msg":"404 Route Not Found"}` | APISIX route matcher | /healthz is a strict GET. |
| **E4** | edge | POST issuance valid cert + valid key + **empty body** | **400** | empty (Spring-Security headers) | dc-tang request parser | Cert + auth pass; dc-tang correctly rejects unparseable body. |
| **E5** | edge | POST directly to `/iso/api/v1/issuance-platform` (the rewritten path APISIX uses internally) | **404** | `{"error_msg":"404 Route Not Found"}` | APISIX route matcher | **Security**: dc-tang's internal path is NOT exposed publicly — cert-validator cannot be bypassed by skipping the public prefix. |

### Two distinct 404 shapes — useful to know which side rejected

- `{"error_msg":"404 Route Not Found"}` — APISIX (no matching route).
- Empty body + Spring-Security headers (`vary: Origin`, `expires: 0`, `pragma: no-cache`, `x-frame-options: DENY`) — dc-tang.

### Detailed cases

The shell variables below are set once and reused:

```bash
# Prereqs (run once per terminal)
openssl pkcs12 -in ~/DevWorkspace/apisix/silmarils/apisix/client-certs/abudhabicommercialbank.p12 \
  -clcerts -nokeys -nodes -passin pass:password 2>/dev/null \
  | openssl x509 -outform PEM > /tmp/adcb.pem
ADCB_CERT=$(jq -sRr @uri < /tmp/adcb.pem)
API_KEY='<PASTE_VALID_ADCB_KEY_HERE>'

# Pacs.009 ISUE template lives at /tmp/pacs009-isue.xml — see §2 Level 3 above.
```

#### H1 — `/healthz` liveness

```bash
curl -i https://silmarils-qa-apisix.takamul.cc/healthz
```

```http
HTTP/2 200
content-type: application/json
server: cloudflare
content-length: 44

{"status":"ok","service":"apisix-silmarils"}
```

---

#### H2 — full pacs.009 ISUE happy path

```bash
# (regenerate dynamic fields in /tmp/pacs009-isue.xml — see §2 Level 3 step b)
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' -H 'Accept: text/xml' \
  -H "X-Forwarded-Client-Cert: $ADCB_CERT" \
  -H "X-LFI-API-KEY: $API_KEY" \
  --data-binary @/tmp/pacs009-isue.xml
```

```http
HTTP/2 200
content-type: text/xml; charset=UTF-8

<SOAP-ENV:Envelope …>
  <SOAP-ENV:Header>
    <MsgTp>admi.098.001.01</MsgTp>
    <Network>MBRIDGE</Network>
    …
  </SOAP-ENV:Header>
  <SOAP-ENV:Body>
    <ns2:Document xmlns:ns2="urn:iso:std:iso:20022:tech:xsd:admi.098.001.01">
      <ns2:Rct>
        <ns2:RctDtls><ns2:ReqHdlg>
          <ns2:OrgnlMsgId>{your-MsgId}</ns2:OrgnlMsgId>
          <ns2:OrgnlMsgNmId>pacs.009.001.12</ns2:OrgnlMsgNmId>
          <ns2:StsCd>RJCT</ns2:StsCd>
          <ns2:Desc>java.lang.reflect.InvocationTargetException SecurityConfiguration class
                    (org.owasp.esapi.reference.DefaultSecurityConfiguration) CTOR threw exception.</ns2:Desc>
        </ns2:ReqHdlg></ns2:RctDtls>
      </ns2:Rct>
    </ns2:Document>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

`StsCd=RJCT` is **finding F12** (dc-tang OWASP ESAPI init bug), not a
gateway issue. HTTP 200 + `OrgnlMsgId` echo + `OrgnlMsgNmId=pacs.009.001.12`
prove the gateway side end-to-end. When dc-tang's ESAPI is fixed,
expect `StsCd=ACCP`.

---

#### S1 — no client cert

```bash
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' --data 'x'
```

```http
HTTP/2 403
content-type: application/json
server: cloudflare

{"error":"client certificate required"}
```

---

#### S2 — self-signed cert with unknown CN

```bash
# Generate a throwaway self-signed cert with an unknown CN
openssl req -x509 -newkey rsa:2048 -days 1 -nodes \
  -keyout /tmp/bad.key -out /tmp/bad.pem \
  -subj '/CN=system@unknownbank.com' 2>/dev/null
BAD_CERT=$(jq -sRr @uri < /tmp/bad.pem)

curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H "X-Forwarded-Client-Cert: $BAD_CERT" \
  --data 'x'
```

```http
HTTP/2 403
content-type: application/json
server: cloudflare

{"error":"certificate validation failed"}
```

---

#### S3 — valid cert, no API key

```bash
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H "X-Forwarded-Client-Cert: $ADCB_CERT" \
  --data 'x'
```

```http
HTTP/2 401
content-type: text/xml; charset=UTF-8

<SOAP-ENV:Envelope …>
  <SOAP-ENV:Header>
    <MsgTp>admi.098.001.01</MsgTp>
    <Network>mBridge</Network>
    …
  </SOAP-ENV:Header>
  <SOAP-ENV:Body>
    <ns2:Document …>
      <ns2:Rct><ns2:RctDtls><ns2:ReqHdlg>
        <ns2:StsCd>RJCT</ns2:StsCd>
        <ns2:Desc>Missing X-LFI-API-KEY header</ns2:Desc>
      </ns2:ReqHdlg></ns2:RctDtls></ns2:Rct>
    </ns2:Document>
  </SOAP-ENV:Body>
</SOAP-ENV:Envelope>
```

---

#### S4 — valid cert, bogus API key

```bash
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H "X-Forwarded-Client-Cert: $ADCB_CERT" \
  -H "X-LFI-API-KEY: lfi_thisIsNotARealKey0000000000000000000000" \
  --data 'x'
```

```http
HTTP/2 401
content-type: text/xml; charset=UTF-8

<SOAP-ENV:Envelope …>
  …
  <ns2:StsCd>RJCT</ns2:StsCd>
  <ns2:Desc>Authentication failed</ns2:Desc>
  …
</SOAP-ENV:Envelope>
```

Same shape as S3, different `Desc` — useful to confirm the auth
filter ran (vs the "missing header" early-exit).

---

#### S5 — valid cert + valid key + stale payload (no Document wrapper)

```bash
# /Users/.../qa-tests/issuance-request-soap.xml has <FICdtTrf> directly
# under <SOAP-ENV:Body>, not wrapped in <Document xmlns="…pacs.009.001.12">.
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H "X-Forwarded-Client-Cert: $ADCB_CERT" \
  -H "X-LFI-API-KEY: $API_KEY" \
  --data-binary @"$HOME/DevWorkspace/apisix/silmarils/apisix/qa-tests/issuance-request-soap.xml"
```

```http
HTTP/2 404
content-length: 0
server: cloudflare
strict-transport-security: max-age=31536000 ; includeSubDomains
vary: Origin
vary: Access-Control-Request-Method
vary: Access-Control-Request-Headers
x-content-type-options: nosniff
x-frame-options: DENY
cache-control: no-cache, no-store, max-age=0, must-revalidate
…
(empty body)
```

This is **dc-tang's** 404 (Spring-Security defaults). dc-tang log
line at this moment:
```
LFI authenticated: lfi-ADCB (scope=MBRIDGE_JISR)
No endpoint mapping found for [SaajSoapMessage FICdtTrf]
```

The stale repo payload is a known issue — use the §2 Level 3
template instead.

---

#### E1 — GET on the POST-only route

```bash
curl -i "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform"
```

```http
HTTP/2 404
content-type: application/json
server: cloudflare

{"error_msg":"404 Route Not Found"}
```

---

#### E2 — POST to an unknown path

```bash
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/totally-fake" \
  -H 'Content-Type: text/xml' --data 'x'
```

```http
HTTP/2 404
content-type: application/json

{"error_msg":"404 Route Not Found"}
```

---

#### E3 — POST on the GET-only `/healthz`

```bash
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/healthz" --data 'x'
```

```http
HTTP/2 404
content-type: application/json

{"error_msg":"404 Route Not Found"}
```

---

#### E4 — empty body with valid cert + valid key

```bash
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H "X-Forwarded-Client-Cert: $ADCB_CERT" \
  -H "X-LFI-API-KEY: $API_KEY" \
  --data ''
```

```http
HTTP/2 400
content-length: 0
server: cloudflare
vary: Origin
…Spring-Security default headers…
(empty body)
```

dc-tang received the request (cert + auth passed) but rejected the
empty body with 400. Useful proof that **the auth pipeline runs
before the body parser** — bad bodies fail differently from auth
failures.

---

#### E5 — try to skip the public prefix (security check)

```bash
# Attempt to hit dc-tang's internal path directly through APISIX,
# bypassing the proxy-rewrite + cert-validator chain.
curl -i -X POST "https://silmarils-qa-apisix.takamul.cc/iso/api/v1/issuance-platform" \
  -H 'Content-Type: text/xml' \
  -H "X-Forwarded-Client-Cert: $ADCB_CERT" \
  -H "X-LFI-API-KEY: $API_KEY" \
  --data-binary @/tmp/pacs009-isue.xml
```

```http
HTTP/2 404
content-type: application/json

{"error_msg":"404 Route Not Found"}
```

**This is the right answer.** The `/iso/api/v1/…` path is *only* an
internal rewrite target; it is **not** a registered APISIX route on
the public hostname. There's no way to reach dc-tang's endpoint
without going through the public `/api/v1/issuance-platform` path,
which means `cert-validator` cannot be bypassed. Treat any 200/401
on this URL as a critical-severity regression.

---

## 4. Troubleshooting tree

| Symptom | Most likely cause |
|---|---|
| `curl: (6) Could not resolve host` | DNS issue on your laptop. Try `dig silmarils-qa-apisix.takamul.cc` — it should return Cloudflare-flavoured IPs (104.21.x.x or 172.67.x.x). |
| `HTTP/2 403` body `{"error":"client certificate required"}` | `X-Forwarded-Client-Cert` header was empty. Most often the `openssl pkcs12` step failed — check that the `.p12` path is correct and that `$(…)` produced output (run `echo "${#CERT}"` after assigning to a var; should be > 1000). |
| `HTTP/2 403` body `{"error":"certificate validation failed"}` | Cert presented but its CN isn't in the gateway's whitelist. Make sure you used a bank `.p12` from `silmarils/apisix/client-certs/` and not a self-signed test cert. |
| `HTTP/2 401` body with `StsCd: RJCT, Desc: Missing X-LFI-API-KEY` | You didn't send the `X-LFI-API-KEY` header (or sent it empty). Add the key. |
| `HTTP/2 401` body with `StsCd: RJCT, Desc: Authentication failed` | Key is rejected by dc-tang — either it's expired, revoked, or never propagated to the gateway DB. Ask for a fresh key. (There's a known controller→gateway sync issue tracked under `MIT-NEXT-controller-gateway-key-sync/`.) |
| `HTTP/2 404` empty body | Body root element doesn't match dc-tang's Spring-WS routing. The pacs.009 payload MUST be wrapped in `<Document xmlns="urn:iso:std:iso:20022:tech:xsd:pacs.009.001.12">`. The template above is correct; don't use `silmarils/apisix/qa-tests/issuance-request-soap.xml` (that file is stale). |
| `HTTP/2 200` body `StsCd: RJCT` with `ESAPI` in Desc | **Gateway is fine** — dc-tang bug F12. Test result is positive from a gateway-verification perspective. |
| Anything 5xx | Capture the full response, attach to the bug — that's an unexpected failure mode, not in the known catalogue. |

---

## 5. Swap to another bank

All 17 banks have a `.p12` in `silmarils/apisix/client-certs/`.
Replace the file name in the `openssl pkcs12 -in …` step:

| Bank | `.p12` file | CN inside the cert | `X-LFI-ID` value set by APISIX |
|---|---|---|---|
| FAB | `firstabudhabibank.p12` | `system@fab.com` | `lfi-FAB` |
| **ADCB** | `abudhabicommercialbank.p12` | `system@adcb.com` | **`lfi-ADCB`** |
| ADIB | `abudhabiislamicbank.p12` | `system@adib.com` | `lfi-ADIB` |
| ENBD | `emiratesnbd.p12` | `system@enbd.com` | `lfi-ENBD` |
| RAK | `rakbank.p12` | `system@rakbank.com` | `lfi-RAK` |
| MASHREQ | `mashreq.p12` | `system@mashreq.com` | `lfi-MASHREQ` |
| BOC | `bankofchina.p12` | `system@bankofchina.com` | `lfi-BOC` |
| ICBC | `icbc.p12` | `system@dxb.icbc.com.cn` | `lfi-ICBC` |
| DIB | `dubaiislamicbank.p12` | `system@dib.ae` | `lfi-DIB` |
| CBD | `commercialbankofdubai.p12` | `system@cbd.ae` | `lfi-CBD` |
| HSBC | `hsbcmiddleeast.p12` | `system@hsbc.com` | `lfi-HSBC` |
| Citi | `citibank.p12` | `system@citi.com` | `lfi-CITI` |
| SCB | `standardcharteredbank.p12` | `system@standardchartered.com` | `lfi-SCB` |
| SIB | `sharjahislamicbank.p12` | `system@sjb.ae` | `lfi-SIB` |
| EIB | `nationalbankoffujairah.p12` | `system@nbf.ae` *(F03 swap — pending owner review)* | `lfi-EIB` |
| NBF | `emiratesislamicbank.p12` | `system@mbf.ae` *(F03 swap — pending owner review)* | `lfi-NBF` |
| CB | `centralbank.p12` | `system@centralbank.com` | `lfi-CB` |

All `.p12` files share the passphrase `password` — silmarils
convention from `apisix/setup-certs.sh`.

The API key is **per-bank**: an ADCB-issued key won't authenticate
as ENBD. Ask the platform team for the right key for the bank you're
testing.

For the payload itself: it's fine to keep the body ADCB-flavoured
(LEI / MmbId / account IDs) when testing other banks, as long as you
only care about gateway plumbing. Content-level validation against
the authenticated LFI is a deeper-layer test that's out of scope for
this smoke.

---

## 6. What to report when filing a bug

Whether the test passes or fails, include in your report:

- The **exact curl** you ran (redact only the `X-LFI-API-KEY` value).
- The **full HTTP response** (status line, all headers, body — `-i`
  already captures this).
- The **`MsgId` you generated** (visible in the SOAP request body
  before sending and echoed back as `OrgnlMsgId` in the response —
  use it to correlate with dc-tang logs).
- Timestamp of the call (UTC).
- Which bank `.p12` you used.

That's enough for either the gateway team (MIT-5211, APISIX side) or
the silmarils application team (dc-tang side) to trace the request
end-to-end.

---

## 7. Reference

- **Architecture diagram + per-layer breakdown**:
  `MIT-5211/task/health-check.md`.
- **Known findings**:
  `MIT-5211/task/FINDING.md` (F01–F12) — especially F12 (the ESAPI
  bug you'll most likely hit as the RJCT body) and F11 (the
  controller→gateway sync issue affecting new key generation).
- **Gateway commit history**:
  `https://github.com/takamulai/silmarils/commits/MIT-5211`.
- **Sibling Cloudflare tunnel PR**:
  `https://github.com/takamulai/eng-infra/commits/MIT-5211`.
