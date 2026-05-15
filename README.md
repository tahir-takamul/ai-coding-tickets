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
