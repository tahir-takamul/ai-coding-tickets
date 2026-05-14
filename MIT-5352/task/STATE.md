# STATE — MIT-5352 mobile-v2 pipeline revamp

> Read this first when resuming. Live operational status.

## Pickup steps for fresh context

1. Read `PLAN.md` for the overall direction.
2. Read `FINDING.md` — especially F01–F04, which shape the artefact
   handoff design.
3. Skim the table below for what's done / in-flight.
4. The legacy pipeline is at
   `iac/cicd-onprem/pipelines/mobile/` in `takamulai/mithril` —
   **do not edit it**. New work goes under
   `iac/cicd-onprem/pipelines/mobile-v2/`.

## Current state (2026-05-14)

| Layer | Status | Notes |
|---|---|---|
| `mobile-v2/` directory | created | `modules/`, `orchestrators/` |
| `mobile-v2/modules/sync-github-to-bitbucket.Jenkinsfile` | done (T01) | Pushes GitHub `staging` → Bitbucket `staging` + all staging-reachable tags. Bitbucket default (`main`) untouched. |
| `mobile-v2/modules/security-scan.Jenkinsfile` | stub (T02) | Echoes TODO; orchestrator wired to call it. Port from legacy is T02. |
| `mobile-v2/modules/android-build.Jenkinsfile` | stub (T03) | Echoes TODO; orchestrator wired. Port is T03. |
| `mobile-v2/modules/ios-build.Jenkinsfile` | stub (T04) | Echoes TODO; orchestrator wired. Blocked on macOS hardware. |
| `mobile-v2/modules/artifact-upload.Jenkinsfile` | stub (T05) | Echoes TODO; orchestrator wired. Port is T05. |
| `mobile-v2/modules/mdm-distribute.Jenkinsfile` | partial (T06) | Real Nexus download (curl + `Nexus-cred`); MDM upload is still echo, blocked on MDM platform. |
| `mobile-v2/orchestrators/bootstrap-orchestrator.Jenkinsfile` | wired end-to-end | Init → Sync → Resolve Version → Security Scans (parallel) → Build Android → Build iOS → Upload Nexus → Distribute to MDM. Fail-safe mode throughout. |
| Jenkins job wiring | not started | Need new folder `Retail-CBDC/00-Mobile/` with `Modules/` + `Orchestrators/` subfolders containing one job each. |
| UAT / PREPROD / PROD orchestrators | not started | After bootstrap chain is complete |

## In-flight task

None — T01 landed; orchestrator + stubs + task specs T02-T06 landed.
Next: pick up T02 (security-scan port) when user is ready.

## Open questions / pending decisions

- **Jenkins folder path** for mobile-v2 jobs — confirmed as
  `Retail-CBDC/00-Mobile/{Modules,Orchestrators}/...`. (Note the
  PascalCase `Modules`/`Orchestrators` segments — child job lookups
  are case-sensitive.)
- **MDM platform** — module is echo-only until platform team picks
  Intune / Workspace ONE / etc.
- **macOS hardware** — `ios-build` stays as a placeholder until
  provisioned.

## Recent changes

- `2026-05-14` — T01: created `mobile-v2/` skeleton, sync module
  (with tags), bootstrap-orchestrator (then named "SIT") with Init +
  Sync + Resolve Version.
- `2026-05-14` — Wired full bootstrap-orchestrator flow: stubs for
  security-scan / android-build / ios-build / artifact-upload, echo
  placeholder for mdm-distribute, T02-T06 task specs landed,
  FINDING.md F01-F04 documented.
- `2026-05-14` — Renamed Jenkins job paths in orchestrator from
  `Retail-CBDC/MobileApp2/modules/...` to
  `Retail-CBDC/00-Mobile/Modules/...` (PascalCase, platform-team
  convention). Docs swept.
- `2026-05-14` — `mdm-distribute` upgraded from pure-echo to real
  Nexus download (curl + `Nexus-cred`) + echo MDM upload. T06.md
  status flipped to `partial`.
- `2026-05-14` — Renamed orchestrator file
  `sit-orchestrator.Jenkinsfile` → `bootstrap-orchestrator.Jenkinsfile`
  (legacy mobile/ called this "SIT" but it builds the bootstrap
  variant — clearer name). All mobile-v2 internal references swept;
  legacy `mobile/orchestrators/sit-orchestrator.Jenkinsfile` paths
  retained as legacy references.
- `2026-05-14` — F05: `environment { … = "${params.X}" }` + `set -u`
  abort on first parameterised run; fixed with defensive
  `${VAR:-}` coercion in the orchestrator's Resolve Version stage.
- `2026-05-14` — F06: sync-github-to-bitbucket skip check was
  comparing local-only tag SHAs against itself (git's shared tag
  namespace bug) — every tag falsely marked "skipped", nothing
  pushed. Fixed by snapshotting Bitbucket's tag set via
  `git ls-remote --tags --refs` and comparing against that. Re-run
  the sync to backfill the ~2060 staging-reachable tags.
