# STATE — controller→gateway `lfi_api_config` sync investigation

> Read this first when resuming. Live operational status.
> Placeholder ticket — rename to the real Jira number once filed.

## Pickup steps for fresh context

1. Read `PLAN.md` for the goal + the silmarils-qa symptom.
2. Cross-reference MIT-5211 `task/FINDING.md` F11 (the drop) and the
   verified 200 artefact under MIT-5211 `task/verified/`.
3. Start with Kafka consumer-group lag (PLAN.md §"Likely areas to
   investigate" #1) — that's the fastest signal.

## Current state (2026-05-18)

| Layer | Status | Notes |
|---|---|---|
| Problem discovered | done | While running pacs.009 ISUE through MIT-5211 APISIX gateway. |
| Workaround applied (silmarils-qa, ADCB only) | done | UPDATE on gateway DB `lfi_api_config` row to use the controller-side hash. dc-tang restarted. HTTP 200 verified. |
| Kafka consumer lag check | not started | First diagnostic step. |
| Root cause analysis | not started | — |
| Sync restored | not started | — |
| Workaround revoked | not started | Need to revoke key `lfi_MOt6n-C9…` once sync is back and a fresh key is propagated. |

## In-flight task

None — the ticket is filed as a placeholder. Pick up when prioritised.

## Open questions

- Did the sync break for **all** lfi_api_config changes (PRIMARY,
  SECONDARY, REVOKE) or only for new INSERTs? Generate a SECONDARY +
  REVOKE to compare propagation timing.
- Is this scoped to `MBRIDGE_JISR` network_scope, or also `RETAIL`?
  Repro on a RETAIL endpoint to scope.
- Did anything else in silmarils-qa fail at the same time? Check
  monitoring dashboards for the 06:00-10:00 UTC window on 2026-05-18.

## Recent changes

- `2026-05-18` — Ticket folder created during MIT-5211 wrap-up. Symptom
  captured in PLAN.md. Workaround documented; same workaround used to
  unblock MIT-5211 final verification.
