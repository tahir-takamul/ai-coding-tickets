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
| `mobile-v2/modules/sync-github-to-bitbucket.Jenkinsfile` | done (T01) | Pushes GitHub `staging` → Bitbucket `staging` + latest 20 staging-reachable tags (tweak `TAG_LIMIT` to change). Bitbucket default (`main`) untouched. |
| `mobile-v2/modules/security-scan.Jenkinsfile` | done (T02) | Full SonarQube + Checkmarx SAST/SCA port from legacy. `BRANCH=staging` / `ENVIRONMENT=bootstrap` defaults + optional `COMMIT_SHA` plumbed into the Sonar checkout. |
| `mobile-v2/modules/android-build.Jenkinsfile` | done (T03) | Full APK-build port from legacy (assembleBootstrapRelease, signing creds, archive). `BUILD_TAG` now uses `RESOLVED_VERSION_LABEL` (orchestrator passes `RESOLVED_VERSION`), falls back to `BRANCH`. |
| `mobile-v2/modules/ios-build.Jenkinsfile` | stub (T04) | Echoes TODO; orchestrator wired. Blocked on macOS hardware. |
| `mobile-v2/modules/artifact-upload.Jenkinsfile` | done (T05) | Full port from legacy (controller-disk archive read → curl PUT to Nexus). Now sets `env.NEXUS_ARTIFACT_URL = uploadPath` so the MDM stage can consume it via `result.buildVariables`. |
| `mobile-v2/modules/mdm-distribute.Jenkinsfile` | partial (T06) | Real Nexus download (curl + `Nexus-cred`); MDM upload is still echo, blocked on MDM platform. |
| `mobile-v2/orchestrators/bootstrap-orchestrator.Jenkinsfile` | wired end-to-end | Init → Sync → Resolve Version → Security Scans (parallel) → Build Android → Build iOS → Upload Nexus → Distribute to MDM. Fail-safe mode throughout. |
| Jenkins job wiring | not started | Need new folder `Retail-CBDC/00-Mobile/` with `Modules/` + `Orchestrators/` subfolders containing one job each. |
| UAT / PREPROD / PROD orchestrators | not started | After bootstrap chain is complete |

## In-flight task

None — every module that can land has landed: T01 sync, T02 scan,
T03 android-build, T05 artifact-upload, T06 download-from-Nexus. T04
ios-build stays a placeholder (macOS hardware), T06 MDM-upload half
stays an echo (MDM platform). Next: Jenkins-side job creation under
`Retail-CBDC/00-Mobile/{Modules,Orchestrators}/...` and a first
end-to-end orchestrator run.

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
  `git ls-remote --tags --refs` and comparing against that.
- `2026-05-14` — Sync now mirrors only the latest 20 staging-reachable
  tags (by creation date) instead of all ~2k. `TAG_LIMIT` in the
  Sync stage is the knob.
- `2026-05-14` — Ported T02/T03/T05 in parallel: security-scan,
  android-build, artifact-upload Jenkinsfiles in mobile-v2 are no
  longer stubs — they carry the legacy logic verbatim plus the
  mobile-v2 deltas (BRANCH/ENV defaults, `COMMIT_SHA` plumbing,
  `RESOLVED_VERSION_LABEL` for BUILD_TAG, `NEXUS_ARTIFACT_URL`
  surfaced via `env`). Only T04 (ios-build) remains a placeholder,
  parked on macOS hardware. Status flips on T02/T03/T05.
