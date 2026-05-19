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

## 3. Troubleshooting tree

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

## 4. Swap to another bank

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

## 5. What to report when filing a bug

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

## 6. Reference

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
