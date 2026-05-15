# AI Coding Tickets — Archive & Convention

> Per-ticket archive of design + planning + operational docs produced during
> Claude Code sessions on Takamul engineering work. The source repo (e.g.
> `takamulai/mithril`, `takamulai/eng-infra`) is where code lands; this repo
> is where the *thinking and runbook* land.

## Directory layout

```
<JIRA-TICKET-NUMBER>/
  task/
    PLAN.md         original rollout plan (frozen pre-execution)
    STATE.md        live operational status — read first when resuming
    T01.md … Tnn.md per-task specs and what shipped
    T-<name>.md     specs delegated to a specific human (e.g. T-Rohit.md)
    FINDING.md      numbered gotchas (Fnn) discovered during execution
    REVERT.md       what was created and how to undo each piece
    REUSE.md        (when applicable) onboarding guide for adopting in another repo
    <SYSTEM>.md     (when applicable) top-level system documentation
                    e.g. SONAR.md, KAFKA.md — entry point for humans
    logs/           apply logs, console snapshots, TF state pre/post, etc.
    done/           (optional) tasks moved here when complete
    verified/       (optional) verification artifacts
```

Example: `MIT-3467/task/SONAR.md`.

## When working on a new ticket as Claude

1. **Create the ticket folder** in this repo: `mkdir <TICKET>/task` and start with `PLAN.md` capturing scope + decisions. Commit early; this repo is the durable record.
2. **Per-task specs (`T01.md`, `T02.md`, …)** follow the template seen in `MIT-3467/`. Each has:
   - `**Status**: pending | DONE` header
   - `**Repo**`, `**Blocks**`, `**Blocked by**` cross-references
   - `## Reason`
   - `## Input` (what to read before starting)
   - `## Output` (concrete edits)
   - `## Acceptance criteria`
   - `## Notes`
3. **`STATE.md` is the resumption entry point.** Update it as work progresses so the *next* Claude session (or a human picking up the ticket) sees current state without re-deriving from commits.
4. **`FINDING.md`** is for gotchas worth carrying forward. Each entry follows: `**What** / **Why it matters** / **How we handle it** / **Do NOT**`. Number them `F01`, `F02`, …
5. **`REVERT.md`** lists everything created: TF resources, KV secrets, K8s objects, ACR images, GHA workflows, etc. Each row has the revert command. This is the safety net.
6. **`logs/`** keeps apply outputs verbatim. Long but invaluable when debugging two months later. Don't trim them.
   - **Exception**: **never commit TF state dumps** (`terraform state pull > foo.json`) to this archive — Terraform state contains backend credentials (Azure Storage Account access keys, GCS/S3 creds, etc.) that GitHub secret-scanning correctly blocks. If a state snapshot is needed for forensics, store it in the cloud bucket itself (e.g. `${TF_BACKEND_BUCKET}/snapshots/...`) and link to it from REVERT.md, don't paste the JSON here.
7. IMPORTANT: Keep on pushing the changes to main, however small the change of plan be. Its a personal local, private repo dedicated for the task

## Working across repos

A single ticket usually spans multiple source repos (e.g. MIT-3467 touches `takamulai/mithril` for pipeline code and `takamulai/eng-infra` for helm/TF). The task/ docs reference both. Commits land in the respective source repos; the task/ docs in this archive are the cross-cutting narrative.

## When resuming a ticket from cold context

Read in this order:

1. `<TICKET>/task/STATE.md` — current operational state + "pickup steps for fresh context".
2. `<TICKET>/task/<SYSTEM>.md` if one exists — top-level system documentation.
3. `<TICKET>/task/FINDING.md` — gotchas relevant to anything you're about to touch.
4. The specific `Tnn.md` you're working on.
5. `REVERT.md` only if you need to roll something back.

## File hygiene

- Use real GitHub-flavored markdown. Tables, code fences, mermaid diagrams.
- Plain prose for context; tables for inventories; bullets for procedures.
- No bloat. Each doc earns its size.
- Cross-link aggressively. `FINDING §F19c` should link to `FINDING.md` and not be re-derived elsewhere.
- Frozen vs. live docs: `PLAN.md` and `T01.md`–`Tnn.md` are frozen once accepted (with status flips); `STATE.md`, `FINDING.md`, `REVERT.md`, `SONAR.md` evolve.

## Indexed tickets

| Ticket | Status | Brief |
|---|---|---|
| `MIT-3467` | DONE (T01–T16; T17 pending) | SonarQube Community Build on AKS + Jenkins integration + GHA PR pipeline. Entry point: `MIT-3467/task/SONAR.md`. |
| `MIT-5211` | T02/T03/T07 DONE; T04 in flight; G8-ask out to owner | APISIX on silmarils-qa AKS: two-layer ansible (15a base / 15b silmarils LFI overlay) behind Cloudflare Tunnel; APISIX terminates mTLS via `X-Forwarded-Client-Cert`. Entry point: `MIT-5211/task/STATE.md`. |
