# Plan — Controller → Gateway DB sync for `lfi_api_config` (event drop)

> **Placeholder ticket folder.** Rename to the real Jira number once filed
> (e.g. `MIT-XXXX/`). Discovered while testing pacs.009 ISUE through the
> MIT-5211 APISIX gateway on 2026-05-18. Cross-link: see MIT-5211
> `FINDING.md` F11.

## Goal

Diagnose and restore the controller→gateway sync for `lfi_api_config`
rows in silmarils-qa (and any other env where it's broken). New keys
generated via dc-tanc's `POST /api/v1/organizations/<orgId>/api-config/.../inbound/generate`
must propagate to dc-tang's gateway DB so the key is accepted at
runtime auth.

## Symptom (2026-05-18 silmarils-qa)

1. `POST .../inbound/generate` returns `HTTP 201` with a fresh
   `apiKey` plaintext (10:42:13 UTC).
2. Controller DB row exists immediately:
   ```sql
   silmarils_qa_controller=> SELECT key_type, status, config FROM lfi_api_config
                            WHERE lfi_org_id='45b3c30b-…ADCB' AND direction='IN';
   PRIMARY | ACTIVE | { keyId: 3d178f66ee8f87d2, keyPrefix: lfi_MOt6n-C9, hashedKey: jW0c…f8s=, expiresAt: 2026-08-16T10:42 }
   ```
3. Gateway DB **does not get the new row** — newest entry remained
   `lfi__Pb00SR6` (created earlier at 05:47 UTC).
4. Subsequent `POST /api/v1/issuance-platform` through APISIX with
   the new plaintext → dc-tang `LfiApiAuthFilter` logs
   `Authentication failed for LFI: lfi-ADCB` because the hash doesn't
   match anything in gateway DB.
5. Earlier in the same day **the sync did work** — gateway DB has
   05:42 / 05:45 / 05:47 entries for ADCB matching contemporaneous
   activity. So this is a regression that happened between ~06:00 and
   ~10:00 UTC on 2026-05-18.

## Workaround applied for MIT-5211 verification

Direct SQL on gateway DB (see MIT-5211 `task/verified/`):
```sql
UPDATE lfi_api_config
   SET config = '{ keyId:…, keyPrefix:…, hashedKey:…, expiresAt:… }'::jsonb,
       updated_at = now(),
       version = version + 1
 WHERE lfi_org_id = '<ADCB org id>'
   AND direction = 'IN' AND key_type = 'PRIMARY'
   AND network_scope = 'MBRIDGE_JISR' AND status = 'ACTIVE';
```
Plus a `kubectl rollout restart deploy/dc-tang -n silmarils-qa` to
drop any in-memory config cache. With those two manual steps applied,
`POST /api/v1/issuance-platform` returns HTTP 200 (auth crosses; see
MIT-5211 F12 for the next downstream issue — separate concern).

**This workaround is for first-bring-up verification only.** Do not
adopt it for any real flow. Revoking + regenerating via the dc-tanc
API would be the legitimate equivalent once sync is restored.

## Likely areas to investigate

1. **Kafka topic for `lfi_api_config` events**
   - dc-tanc env (`10-silmarils-controller.yaml`) shows
     `KAFKA_BOOTSTRAP_SERVERS: kafka.data-silmarils.svc:9092`,
     `KAFKA_TOPIC_PRODUCER: qa.controller.gateway`-ish (verify
     exact topic name — see playbook `KAFKA_TOPIC_*` env vars).
   - dc-tang env shows the matching `KAFKA_TOPIC_CONSUMER`. Confirm.
   - Check consumer-group lag:
     ```bash
     kubectl -n data-silmarils exec kafka-0 -- \
       kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
       --describe --group dc-tang-…
     ```
   - Suspect lag is non-zero and frozen at ~05:47 UTC; that would
     explain the symptom cleanly.

2. **Producer side** — dc-tanc's `generate` endpoint may have stopped
   publishing events. Check dc-tanc logs around 06:00-10:00 UTC for
   the inflection point (any error / exception, container restart,
   config change, GC pause).

3. **Consumer side** — dc-tang's `KafkaConfigEventListener` (or
   whatever class name) may have stopped polling. Check
   `kubectl -n silmarils-qa logs deploy/dc-tang --since=12h` for
   consumer-related errors / disconnects.

4. **Schema / serialization mismatch** — if either side was deployed
   with a new image and the event schema changed (Avro / JSON),
   the consumer might be silently dropping mismatched events.
   Check the image tags of both dc-tanc and dc-tang against the
   `lfi_api_config` event schema version each side expects.

5. **Manual retrigger** — if there's an admin endpoint on dc-tanc to
   "republish all active keys" or to "rebuild gateway DB from
   controller DB" (some silmarils services have these for backfill),
   try it as a quick recovery.

## What "done" looks like

- Root cause documented in `FINDING.md` (e.g. F01 for this ticket).
- Sync verified by:
  1. Generate a fresh key via dc-tanc.
  2. Within 60s, gateway DB row appears with same `keyPrefix` / `hashedKey`.
  3. `POST /api/v1/issuance-platform` with the new plaintext → HTTP 200,
     `StsCd != "RJCT (Authentication failed)"` (a different RJCT for
     business reasons is OK — that's MIT-5211 F12 territory).
- Any backlog of dropped events between ~06:00 and ~10:00 UTC on
  2026-05-18 either replayed or written off (depending on how many).
- Re-revoke the temporary `lfi_MOt6n-C9…` key (created during
  MIT-5211 verification) to clean state.

## Inputs (read first)

- MIT-5211 `task/FINDING.md` F11 (sync drop) + F12 (ESAPI bug).
- MIT-5211 `task/verified/T08-tunnel-pacs009-isue-adcb-200.xml` —
  the HTTP 200 proof artefact after the workaround.
- `silmarils/iac/eng-infra/shared-k8s/ansible/playbooks/10-silmarils-controller.yaml`
  and `11-silmarils-gateway.yaml` — the Kafka topic env vars on
  each side.
- `silmarils/backend/controller-gateway/platform/services-security/`
  src/main/kotlin — `LfiApiAuthFilter`, `InboundApiKeyValidator`,
  `LfiApiConfigService` (config reader on the gateway side). Find
  the corresponding Kafka listener.

## Out of scope

- MIT-5211 F12 (dc-tang OWASP ESAPI init exception when processing
  the pacs.009 body). Tracked under MIT-5211; only mentioned here
  because it's the *next* layer that fails once sync is restored.
- API key rotation policy / TTL semantics. Independent ticket.
- Building any new admin tooling — the dc-tanc admin endpoints exist
  already.
