# Plan — Mobile pipeline revamp (`mobile-v2/`)

> Master plan for MIT-5352. Live state lives in `STATE.md`; gotchas in
> `FINDING.md`. Per-task specs are `T01.md`, `T02.md`, … and are frozen
> once accepted.

## Goal

Rebuild the on-prem mobile CI/CD pipeline under a **new** directory
`iac/cicd-onprem/pipelines/mobile-v2/` in `takamulai/mithril`. The
existing `mobile/` tree is left untouched — `mobile-v2/` is the green-field
replacement.

The new design starts from the **bootstrap-orchestrator** (the
legacy mobile/ pipeline called this "SIT" but it built the
bootstrap-flavour APK — mobile-v2 renames it for clarity) and grows
downward one step at a time. Each step lands as its own `Tnn.md`.

## Scope split

| Component | Repo | Path |
|---|---|---|
| Mobile-v2 Jenkinsfiles (modules + orchestrators) | **this repo** (mithril) | `iac/cicd-onprem/pipelines/mobile-v2/` |
| Task docs / runbook | **ai-coding-tickets** | `MIT-5352/task/` |

The legacy `mobile/` tree at `iac/cicd-onprem/pipelines/mobile/` is
**read-only** for this ticket — do not edit it. Jenkins jobs that point
at `mobile/` keep working unchanged.

## Architecture (orchestrator chain)

```
Bootstrap ─▶ (build job:) sync-github-to-bitbucket
          ─▶ (build job:) … all wired stages
UAT       ─▶ (later)
PREP ─▶ (later)
PROD ─▶ (later)
```

Modules are standalone Jenkins jobs in `mobile-v2/modules/`; orchestrators
in `mobile-v2/orchestrators/` call them via `build job:`. Matches the
existing mobile pattern.

## Source-of-truth flow

```
GitHub takamulai/mithril               Bitbucket retailcbdc
─────────────────────────────          ──────────────────────
branch: staging       ──[sync]──▶     branch: staging   (new — created on first sync)
tags  : reachable from staging        same tag names (mirrored)
                                      branch: main      (default — untouched, used by legacy mobile/)
```

GitHub `staging` is the dev source of truth. Every commit on `staging`
is tagged with a release version. The sync module pushes the staging
HEAD to a `staging` branch on Bitbucket (created on first run) and
mirrors all staging-reachable tags onto Bitbucket. Bitbucket's default
branch (`main`, used by the legacy `mobile/` pipeline and others)
is **not** touched by mobile-v2. From the Bitbucket side, tag names are
identical to GitHub.

## Decisions (user-confirmed)

| Area | Choice |
|---|---|
| New tree location | `iac/cicd-onprem/pipelines/mobile-v2/` (greenfield, parallel to `mobile/`) |
| First env | bootstrap (legacy called this SIT) |
| First step | GitHub→Bitbucket sync |
| Sync source / target | GitHub `staging` → Bitbucket `staging` (created on first run; Bitbucket default `main` untouched) |
| Tags | Latest 30 tags reachable from `staging` (by creation date) — older tags are historical |
| Sync style | Force-with-lease branch push + tag-by-tag push of staging-reachable tags |
| Version selection | `VERSION_TAG` param; empty → check out HEAD of Bitbucket `staging` (no tag lookup) |
| Sync location | Standalone module job, orchestrator calls via `build job:` |

## Critical anchor files (existing)

- `iac/cicd-onprem/pipelines/modules/sync-github-to-bitbucket.Jenkinsfile` —
  legacy sync used by `mobile/`. Shape to mirror, with two changes:
  (a) include staging-reachable tags, (b) module lives under `mobile-v2/`.
- `iac/cicd-onprem/pipelines/mobile/orchestrators/sit-orchestrator.Jenkinsfile` —
  shape reference for the new `mobile-v2` bootstrap-orchestrator
  (parameters, init stage, post block).

## Tasks

| Task | Title | Status |
|---|---|---|
| `T01` | Initialise mobile-v2 + bootstrap-orchestrator sync step (Init → Sync → Resolve Version) | DONE |
| `T02` | Port `security-scan` module + add `ENABLE_SECURITY_SCANS` toggle | pending |
| `T03` | Port `android-build` module + wire `Build Android` stage | pending |
| `T04` | Port `ios-build` placeholder + wire `Build iOS` stage | pending (blocked on macOS hardware) |
| `T05` | Port `artifact-upload` module + wire `Upload Android Artifact` stage | pending |
| `T06` | Create `mdm-distribute` module (Nexus → MDM, echo-only) + wire `Distribute to MDM` stage | pending (echo today; real impl blocked on MDM platform) |

Note: the bootstrap-orchestrator is **wired end-to-end now** with stub
modules in `mobile-v2/modules/`. Each `Tnn` task above turns one stub
into the real ported / implemented module. The orchestrator itself does
not need re-touching as those tasks land — only the corresponding
module Jenkinsfile.

## Bootstrap orchestrator stage map (post-T01 + wiring pass)

```
Init
  └─ Sync GitHub → Bitbucket                   (T01 — module ported)
  └─ Resolve Version                           (T01 — inline in orchestrator)
  └─ Security Scans (parallel, gated by ENABLE_SECURITY_SCANS)
        ├─ SonarQube                           (T02)
        ├─ Checkmarx SAST                      (T02)
        └─ Checkmarx SCA                       (T02)
  └─ Build Android        (when OS in [Android,Both])     (T03)
  └─ Build iOS            (when OS in [iOS,Both])         (T04)
  └─ Upload Android       (when ENABLE_UPLOAD + Android SUCCESS) (T05)
  └─ Distribute to MDM    (when ENABLE_MDM_DISTRIBUTE + UPLOAD SUCCESS) (T06)
```

## Bootstrap orchestrator parameters

| Name | Type | Default | Purpose |
|---|---|---|---|
| `VERSION_TAG` | string | `''` | Release tag to build; empty ⇒ bitbucket/staging HEAD |
| `OS` | choice (Android/iOS/Both) | `Android` | Target platform(s) |
| `ENABLE_SECURITY_SCANS` | boolean | `true` | Skip scans when false (operator opt-out for hotfix runs) |
| `ENABLE_UPLOAD` | boolean | `true` | Skip Nexus upload when false |
| `ENABLE_MDM_DISTRIBUTE` | boolean | `true` | Skip MDM stage when false |

## Out of scope (for now)

- UAT / PREPROD / PROD orchestrators
- Builds, scans, uploads (these are next-step specs)
- Migrating Jenkins jobs to point at `mobile-v2/` paths (will happen once
  the whole orchestrator chain is in place)
