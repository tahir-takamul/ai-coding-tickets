# FINDING — MIT-5352

> Gotchas worth carrying forward. Each entry: **What / Why it matters /
> How we handle it / Do NOT**. Numbered `F01`, `F02`, … and linked from
> other docs.

---

## F01 — `Copy Artifact` plugin is **not installed** on this Jenkins

**What.** The Jenkins controller at `dsoorchestration.cbuae.gov` does
**not** have the `Copy Artifacts` plugin installed. The legacy mobile
artifact-upload module documents this explicitly at
`iac/cicd-onprem/pipelines/mobile/modules/artifact-upload.Jenkinsfile:75-78`:

> Copy Artifact plugin is not installed on this Jenkins.

**Why it matters.** This is what shapes the entire artifact-handoff
design. Without `copyArtifacts`, a job on agent A cannot pull a file
archived by a job on agent B over the network. The workaround in
legacy is to **read the archive directly off the Jenkins controller's
local disk** (see [[F03]]) — which forces the upload job to run on the
controller itself (see [[F02]]).

**How we handle it.** Mobile-v2 inherits the same constraint for now.
The orchestrator passes `SOURCE_JOB` (`result.fullProjectName`) +
`SOURCE_BUILD_NUMBER` (`result.number`) from the build module to the
upload module, which translates them into a controller-local archive
path and `cp`s the file out.

**Do NOT.**
- Do NOT pin `artifact-upload` to a build agent label (e.g.
  `cbdc_retail`) — it loses filesystem access to `$JENKINS_HOME` and
  silently fails to find the archive. Stay on `agent { label 'Jenkins' }`.
- Do NOT change the orchestrator to `propagate: true` on `android-build`
  until you also propagate `result.fullProjectName` / `result.number`
  somewhere durable — these are only available when `wait: true`.
- Do NOT assume `archiveArtifacts` stores files on the build agent.
  It uploads back to the controller; `cleanWs` wipes the agent copy.

**If/when `copyArtifacts` gets installed** the controller-disk read
collapses to a single `copyArtifacts projectName: SOURCE_JOB, selector:
specific(SOURCE_BUILD_NUMBER)` call and the upload job can run on any
agent. Worth raising with the platform team.

---

## F02 — Android build and artifact upload run on **different agents**

**What.** Despite the README implying a single pipeline, the two
modules run on different Jenkins agents:

| Module | `agent { label }` | Where |
|---|---|---|
| `android-build` | `cbdc_retail` | Build VM (JDK 20, Node 18, Android SDK, proxy config) |
| `artifact-upload` | `Jenkins` | The Jenkins controller itself |

**Why it matters.** This is the direct consequence of [[F01]]. The
upload job needs `$JENKINS_HOME` filesystem access; only the controller
has it. The build job needs the Android toolchain; only `cbdc_retail`
has it. Splitting them is forced.

**How we handle it.** Mobile-v2 keeps the same split. The orchestrator
itself can stay on `cbdc_retail` (it already does because the sync +
resolve-version stages need a Bitbucket workspace) and uses `build job:`
to dispatch each module on its correct agent.

**Do NOT.**
- Do NOT collapse build + upload into one job on `cbdc_retail` without
  also rewriting the upload to push to Nexus *directly from the build
  agent* (bypassing the controller archive). This is a viable
  simplification but a deliberate one — call it out before doing it.
- Do NOT assume that "running on the controller" means the controller
  is doing build work. The `Jenkins` label is the controller's *own*
  executors, used only for lightweight orchestration and disk reads.

---

## F03 — Artifact handoff: `archiveArtifacts` → controller disk → `cp`

**What.** Step-by-step flow that the legacy SIT pipeline uses to move
the APK from the build agent to Nexus:

1. `android-build` (on `cbdc_retail`) runs `assembleBootstrapRelease`,
   producing `wallet/android/app/build/outputs/apk/...apk` in the
   workspace.
2. The `Archive` stage runs `archiveArtifacts artifacts: '…/*.apk'`.
   Despite the build running on an agent, this uploads the file back
   to the **controller** at
   `$JENKINS_HOME/jobs/<job-path>/builds/<N>/archive/...`.
3. `post { always } cleanWs` then wipes the agent workspace (keeping
   only `node_modules`/`.gradle`/`build` as cache). The APK is **gone
   from the agent** at this point.
4. The orchestrator captures `result.fullProjectName` and `result.number`
   from the `android-build` step and passes them to `artifact-upload`.
5. `artifact-upload` (on the controller) constructs the on-disk archive
   path (see [[F04]]) and `cp`s the file into its own workspace's
   `artifacts/` folder.
6. From `artifacts/`, it uploads to Nexus via `curl`.

**Why it matters.** Two things flow from this:
- The build's `buildDiscarder` retention (`numToKeepStr: '20'` in
  legacy) is also the **artifact retention** — once a build is
  discarded, the archive is gone, and re-running just the upload step
  for an old `SOURCE_BUILD_NUMBER` will fail.
- `archiveArtifacts` is the only mechanism that persists the artifact
  beyond the agent workspace. If a future module skips it, downstream
  upload has nothing to read.

**How we handle it.** Mobile-v2 keeps the same shape. The MDM-distribute
module ([[T06]]) is the *second* consumer of the same artifact path —
it must accept the same `SOURCE_JOB` + `SOURCE_BUILD_NUMBER` (or a
Nexus URL, once the upload step has run) so it can locate the file.

**Do NOT.**
- Do NOT shrink the build's `numToKeepStr` below what the typical
  ramp-of-uploads will reference. If MDM uploads happen days after the
  Nexus upload, the archive will already have been pruned — fine
  because MDM should read from Nexus, not from the controller archive.
  But if Nexus upload itself is delayed, the archive read can race
  the discarder.

---

## F04 — `SOURCE_JOB` path → controller filesystem mapping

**What.** The `SOURCE_JOB` value (e.g.
`Retail-CBDC/MobileApp/modules/android-build`) is the Jenkins
**full project name**, with each `/` representing a folder boundary.
On the controller's filesystem this maps to
`$JENKINS_HOME/jobs/Retail-CBDC/jobs/MobileApp/jobs/modules/jobs/android-build/`,
where every original `/` becomes `/jobs/`. The exact translation in
the legacy upload module:

```groovy
def jobFsPath  = 'jobs/' + params.SOURCE_JOB.replace('/', '/jobs/')
def archiveDir = "${jenkinsHome}/${jobFsPath}/builds/${params.SOURCE_BUILD_NUMBER}/archive"
```

**Why it matters.** If the Jenkins folder layout changes (e.g.
legacy `MobileApp` vs. mobile-v2's `00-Mobile/Modules`), the **only**
thing that needs to adapt is what the orchestrator passes — the upload
module's translation logic is generic. As long as the orchestrator
passes `result.fullProjectName` literally (and doesn't try to hardcode
the project name), the move is free.

**How we handle it.** Mobile-v2 orchestrator continues to pass
`result.fullProjectName` and `result.number` opaquely. The
upload module re-uses the same translation. The current mobile-v2
Jenkins-folder layout is `Retail-CBDC/00-Mobile/Modules/...` /
`Retail-CBDC/00-Mobile/Orchestrators/...` — confirmed by the platform
team. PascalCase matters: `Modules` and `Orchestrators` (not
lowercase) — child job lookups are case-sensitive.

**Do NOT.**
- Do NOT hardcode `MobileApp` or `00-Mobile` in the upload module —
  always derive from `params.SOURCE_JOB`.
- Do NOT manually URL-encode the path or strip leading slashes — the
  legacy `replace('/', '/jobs/')` recipe is exactly what Jenkins itself
  uses internally; deviating from it is the way to silent breakage.

---

## F05 — `environment { FOO = "${params.X}" }` + `set -u` = first-run abort

**What.** Declarative pipelines that route params into shell via the
top-level `environment {}` block:

```groovy
environment {
    VERSION_TAG_INPUT = "${params.VERSION_TAG}"
}
```

…look like they should reliably surface `VERSION_TAG_INPUT` as an
exported env var inside `sh '…'` blocks. They don't. When
`params.VERSION_TAG` is the empty default (declared
`defaultValue: ''`), Jenkins sometimes does not export the env var at
all on the first parameterised run — the var is **unset** in the
shell, not empty. Combined with `set -u`, the next reference aborts:

```
script.sh.copy: line 17: VERSION_TAG_INPUT: unbound variable
```

Observed on the bootstrap-orchestrator's `Resolve Version` stage,
first-run after the job was newly parameterised.

**Why it matters.** Every shell-level fetch of a Jenkins param value
is suspect. The bug is silent until `set -u` (which all our shells
use, for fail-fast) trips on the unbound reference.

**How we handle it.** Defensive coercion at the top of the shell:

```sh
set -eu
VERSION_TAG_INPUT="${VERSION_TAG_INPUT:-}"
# safe to reference ${VERSION_TAG_INPUT} from here on
```

`${FOO:-}` expands to empty if `FOO` is unset *or* empty — robust
against both the missing-export case and the genuine-empty case.

Alternative: wrap the `sh` call in `withEnv(["FOO=${params.X ?: ''}"])`
inside a `script {}` block — guarantees the var is exported. More
verbose; same outcome. Use this if you need a non-empty default.

**Do NOT.**
- Do NOT trust `environment { FOO = "${params.X}" }` alone when the
  shell uses `set -u`. Always defensively default at the top of the
  `sh '…'` body.
- Do NOT remove `set -eu` from existing shells to "fix" this — `-eu`
  is the only thing that surfaces typos and unbound refs in pipeline
  shells; lose it and you'll silently build the wrong artefact.
- Do NOT change `set -eu` to `set -e`; you lose the unbound-var
  guard but keep error-exit, which makes the next bug harder to find.

---

## F06 — `git rev-parse refs/tags/X` ignores which remote a tag came from

**What.** Git stores tags in a single, flat, **remote-agnostic**
namespace at `refs/tags/<name>`. Unlike branches (`refs/remotes/<remote>/<name>`),
tags don't get a per-remote prefix when you fetch them. So after:

```sh
git fetch github --tags
git fetch bitbucket --tags
```

…the workspace has exactly one `refs/tags/v1.0.0.199`, and
`git rev-parse refs/tags/v1.0.0.199` returns the SHA from **whichever
remote pushed that tag in last**. You cannot tell from local state
whether Bitbucket actually has the tag.

This bit the mobile-v2 sync's idempotency check on the first real run:
the loop was structured as

```sh
GH_TAG_SHA=$(git rev-parse "refs/tags/${tag}")
BB_TAG_SHA=$(git rev-parse "refs/tags/${tag}" 2>/dev/null || echo "")
if [ "${GH_TAG_SHA}" = "${BB_TAG_SHA}" ]; then ... skip ... fi
```

Both lookups returned the same local SHA, so every tag was "skipped"
and **nothing pushed** — even though Bitbucket genuinely had zero tags.
Symptom: `Tags skipped: 2060, Tags pushed: 0` and the Bitbucket Tags
view stays empty.

**Why it matters.** This is the only way to detect "already mirrored"
without a network call per tag. Anyone tweaking the sync logic later
will reflexively reach for the same `rev-parse`-then-compare pattern
and reintroduce the bug.

**How we handle it.** Snapshot Bitbucket's tag state via one
`git ls-remote --tags --refs` call, write to a temp file, look up
each tag with awk:

```sh
BB_TAGS_FILE="$(mktemp)"
trap 'rm -f "${BB_TAGS_FILE}"' EXIT
git ls-remote --tags --refs "${BB_REMOTE}" 2>/dev/null \
    | awk '{ sub("refs/tags/", "", $2); print $2, $1 }' \
    > "${BB_TAGS_FILE}"

# per tag:
BB_TAG_SHA=$(awk -v t="${tag}" '$1 == t {print $2; exit}' "${BB_TAGS_FILE}")
```

`--refs` suppresses annotated-tag peel lines (`<sha>\trefs/tags/X^{}`)
so the SHA we compare against is the tag-object SHA, exactly what
`git rev-parse refs/tags/X` returns locally.

**Do NOT.**
- Do NOT use `git rev-parse refs/tags/<name>` to determine the state
  of any specific remote — it always returns local state.
- Do NOT try to work around this with `git fetch <remote> --tags` +
  another rev-parse — the fetch updates the same local namespace,
  not a remote-scoped one.
- Do NOT swap to `git push --tags` to "push everything" instead of
  filtering — that ignores the staging-reachable filter and would
  drag the TestFlight/feature-branch tags onto Bitbucket
  (the whole reason this loop is filtered in the first place).

Related: this is the kind of network-touching state check that
[[F01]] (`Copy Artifacts` missing) would also benefit from — both
findings document "you can't infer remote state from local refs".

---
