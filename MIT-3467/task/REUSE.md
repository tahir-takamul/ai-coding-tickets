# Reusing the SonarQube PR Scan workflow in another takamulai repo

The mithril repo hosts a reusable GHA workflow + composite action that:
1. Detects which components a PR changed.
2. Triggers a Jenkins `sonar-pr` wrapper job for each changed component.
3. Polls Jenkins for completion.
4. Reads Sonar Web API for metrics + new-code data.
5. Posts a per-component GitHub Check + idempotent PR summary comment +
   inline review comments on issues that intersect the PR diff.

This doc shows what a new repo (e.g. `silmarils`) needs to wire up.

## What's reusable, what's repo-specific

| Layer                      | Lives in        | Per-repo? |
|---------------------------|-----------------|-----------|
| Composite action          | `mithril/.github/actions/sonar-pr-report` | No — shared |
| Reusable workflow         | `mithril/.github/workflows/sonar-scan-reusable.yml` | No — shared |
| Caller workflow           | each consuming repo's `.github/workflows/sonar-scan-pr.yml` | Yes |
| Jenkins JCasC job + token | `eng-infra/cicd/k8s/jenkins/values.yaml` | Yes |
| GHA secrets               | takamulai org-level (preferred) or per-repo | Org-shared |

## Step-by-step

### 1. Add the JCasC job for the new repo

Edit `eng-infra/cicd/k8s/jenkins/values.yaml`. Add a new block alongside
`mithril-folder-sonar-pr-job` (the seed for mithril's wrapper), with the
repo name swapped through:

```yaml
silmarils-folder-sonar-pr-job: |
  jobs:
    - script: >
        pipelineJob('silmarils/modular-jobs/sonar-pr') {
          description('PR-time SonarQube scan dispatcher for silmarils — webhook-triggered from GHA.')
          parameters {
            stringParam('PIPELINE_BRANCH', 'main', 'Git branch for the Jenkinsfile')
            stringParam('COMPONENT', 'service-a', 'component to scan')
            stringParam('COMMIT_SHA', '', 'PR head commit SHA')
            stringParam('PR_NUMBER', '', 'GitHub PR number')
            stringParam('PR_BRANCH', 'main', 'PR base branch (project key suffix)')
          }
          properties {
            pipelineTriggers {
              triggers {
                genericTrigger {
                  genericVariables {
                    genericVariable { key('COMPONENT');  value('\$.component') }
                    genericVariable { key('COMMIT_SHA'); value('\$.commit_sha') }
                    genericVariable { key('PR_NUMBER');  value('\$.pr_number') }
                    genericVariable { key('PR_BRANCH');  value('\$.pr_branch') }
                  }
                  token('silmarils-sonar-pr-token')        # ← unique per repo
                  causeString('Triggered by GHA for PR#\$PR_NUMBER component=\$COMPONENT')
                  printContributedVariables(false)
                  printPostContent(false)
                }
              }
            }
          }
          definition {
            cpsScm {
              scm {
                git {
                  remote {
                    url('git@github.com:takamulai/silmarils.git')   # ← consuming repo
                    credentials('github-ssh')
                  }
                  branch("\$PIPELINE_BRANCH")
                }
              }
              scriptPath('iac/eng-infra/cicd/pipelines/modules/sonar-pr.Jenkinsfile')   # adjust if pipelines live elsewhere
            }
          }
        }
```

Also add the analogous `silmarils-folder-sonar-scan-job` for the inner
`sonar-scan` worker. Both Jenkinsfiles (`sonar-pr.Jenkinsfile`,
`sonar-scan.Jenkinsfile`) can be copied from `mithril/iac/eng-infra/cicd/pipelines/modules/`
into the consuming repo at the same path — they're parameterized.

Apply: `./deploy.sh cloud playbooks/01-jenkins.yml` (eng-infra/cicd/ansible).

### 2. Add org-level GHA secrets (one-time, takamulai)

If not already present at the org level, promote these from a repo where
they exist:

| Secret                     | Source                                              |
|----------------------------|-----------------------------------------------------|
| `CF_ACCESS_CLIENT_ID`      | Cloudflare Zero Trust → Access → Service Auth      |
| `CF_ACCESS_CLIENT_SECRET`  | same                                                |
| `JENKINS_API_USER`         | a SAML user with Job/Read on `<repo>/modular-jobs/*` |
| `JENKINS_API_TOKEN`        | API token for that user (Jenkins UI → user → API Tokens) |
| `SONARQUBE_READ_TOKEN`     | Sonar user token with Browse on the projects        |

GitHub: **Org settings → Secrets and variables → Actions → New organization secret**.
Set Repository access to "Selected repositories" and add the new repo (or "Private repositories").

The CF Access service token also needs to be authorized on the Jenkins
Access app (already done for the existing token). For Sonar API access
specifically, ensure the same service token is allowed on the
`sonarqube.takamul.cc` Access app (terraform resource
`cloudflare_zero_trust_access_policy.cicd_service_token_sonar`).

### 3. Add the caller workflow in the new repo

Create `.github/workflows/sonar-scan-pr.yml`:

```yaml
name: SonarQube PR Scan

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main, staging]
    paths:
      - 'service-a/**'                # ← repo-specific
      - 'service-b/**'
      - '.github/workflows/sonar-scan-pr.yml'

permissions:
  contents: read
  checks: write
  pull-requests: write

concurrency:
  group: sonar-scan-${{ github.event.pull_request.number }}-${{ github.sha }}
  cancel-in-progress: true

jobs:
  detect:
    runs-on: ubuntu-latest
    outputs:
      components: ${{ steps.matrix.outputs.components }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            service-a: ['service-a/**']
            service-b: ['service-b/**']
      - id: matrix
        shell: bash
        run: |
          echo 'components=${{ steps.filter.outputs.changes }}' >> "$GITHUB_OUTPUT"

  scan:
    needs: detect
    if: needs.detect.outputs.components != '[]' && needs.detect.outputs.components != ''
    uses: takamulai/mithril/.github/workflows/sonar-scan-reusable.yml@main
    with:
      components:            ${{ needs.detect.outputs.components }}
      project_key_prefix:    silmarils                       # ← repo-specific
      jenkins_webhook_token: silmarils-sonar-pr-token        # ← repo-specific
    secrets: inherit
```

### 4. Cross-repo composite action caveat

The reusable workflow uses `uses: ./.github/actions/sonar-pr-report`. The path is
relative to the calling repo's checkout — **so it works for mithril (where
the action lives) but not for other repos**. Two ways to make this work:

**Option A (recommended) — extract the composite action to its own repo**

Move `.github/actions/sonar-pr-report/` to `takamulai/gha-sonar-pr-report`
(a new dedicated repo). Update the reusable workflow to:

```yaml
uses: takamulai/gha-sonar-pr-report@v1
```

This is the long-term clean answer. Tag versions semver-style for stability.
Mithril's caller works unchanged. Other repos start working immediately.

**Option B (workaround) — explicit checkout in the consuming repo's caller**

Add a step that checks out the mithril repo before calling the reusable.
Cumbersome. Useful only as a stopgap.

Until A is done, **only mithril can use the reusable workflow**. Any other
repo would need to copy the composite action into their own
`.github/actions/sonar-pr-report/` and call it directly (skipping the
reusable workflow), OR run option A first.

### 5. Sonar project naming + cleanup

The composite action constructs `${prefix}-${component}-${pr_branch}` as the
Sonar project key. With the trigger scoped to `[main, staging]`, each consuming
repo's Sonar projects are exactly `${prefix}-${component}-main` and
`${prefix}-${component}-staging` — 2× components total. No sprawl, no per-PR
cleanup needed.

### 6. First test

Open a draft PR in the new repo that touches one of the watched paths and
targets `main`. The workflow should:
- Fire on `pull_request: opened` for the right base branch.
- Trigger one Jenkins build per changed component.
- Post a GitHub Check + PR summary comment per component within ~5-15 min
  (depending on scan duration).

Common first-run issues:
- **403 from CF Access** → service token not authorized on the new app
  (Jenkins or Sonar). Check CF Zero Trust → Access → Applications.
- **401 from Jenkins API** → `JENKINS_API_USER` doesn't have access to the
  new repo's `<repo>/modular-jobs/*` folder in Jenkins. Either grant via
  JCasC role-based-auth or use a different user.
- **No matrix jobs run** → dorny paths-filter didn't detect changes. Verify
  the `paths` block in the caller workflow matches actual file paths in the
  PR.
- **Sonar API returns project not found** → first scan for this project key
  hasn't completed yet. The Sonar project is auto-created on first scan; the
  composite action gracefully degrades to a no-metrics summary until then.
