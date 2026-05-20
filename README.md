# ai-coding-tickets

Per-Jira-ticket archive of design, planning, and operational documentation produced during Claude Code sessions on Takamul engineering work.

## Layout

```
<TICKET-NUMBER>/task/
  PLAN.md, STATE.md, T01.md…Tnn.md, FINDING.md, REVERT.md,
  REUSE.md (optional), <SYSTEM>.md (optional), logs/
```

Example: [`MIT-3467/task/SONAR.md`](MIT-3467/task/SONAR.md).

## Convention details

See [`CLAUDE.md`](CLAUDE.md) — the convention reference used by Claude Code sessions and any human picking up a ticket.

## Indexed tickets

| Ticket | Status | Brief |
|---|---|---|
| [`MIT-3467`](MIT-3467/task/STATE.md) | DONE (T01–T16; T17 pending) | SonarQube Community Build on AKS + Jenkins integration + GHA PR-time pipeline. Entry point: [`SONAR.md`](MIT-3467/task/SONAR.md). |
| [`MIT-5211`](MIT-5211/task/STATE.md) | **DONE — live on silmarils-qa** (2026-05-16) | APISIX on silmarils-qa AKS: two-layer ansible (15a base / 15b silmarils LFI overlay) behind Cloudflare Tunnel; APISIX terminates mTLS via `X-Forwarded-Client-Cert`. End-to-end verified at `silmarils-qa-apisix.takamul.cc`. |
| [`MIT-NEXT-controller-gateway-key-sync`](MIT-NEXT-controller-gateway-key-sync/task/STATE.md) | placeholder — pending Jira number | Surfaced during MIT-5211 verification: dc-tanc → dc-tang `lfi_api_config` sync stopped propagating new inbound-key rows from ~06:00 UTC on 2026-05-18. Likely Kafka consumer-group lag. |
| [`MIT-5236`](MIT-5236/task/STATE.md) | partial — quiet path live, active path blocked by GH App bypass limitation | Sync wallet `versionName` / `CFBundleShortVersionString` / `package.json.version` to `/VERSION` from inside the release orchestrator. Works when sources are already in sync; fails on `/VERSION` bumps because GitHub Apps cannot bypass `pull_request` or `required_status_checks` ruleset rules even with full permissions. See [`FINDING.md`](MIT-5236/task/FINDING.md) F01. |
