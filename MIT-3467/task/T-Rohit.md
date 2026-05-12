# T-Rohit — One-time Terraform apply by `rohit.verma@takamul.ai`

**Status**: pending — sent to Rohit on 2026-05-06
**Why this exists**: T02 creates an Azure Entra SSO app via the `azuread` Terraform provider. That requires the `Application Administrator` Entra role. `mohd.tahir@takamul.ai` doesn't have it; `rohit.verma@takamul.ai` does. Only Global Admins (Water / Mahmoud / Adm-Mahmoud) can grant the role to others, and `Application Administrator` cannot self-grant. Fastest unblock: Rohit runs the apply on his own machine and sends back two outputs.

## Forwarded message (copy-paste-ready)

> **Context**: Setting up SonarQube on the engineering CI/CD cluster. Need to register an Azure Entra SSO app via Terraform, which requires the `Application Administrator` role. `mohd.tahir@takamul.ai` doesn't have it; you do. The Terraform code is already pushed — please run it on your machine and send back two values.

### Step 1 — Clone the repo (skip if you already have it)

```bash
git clone git@github.com:takamulai/eng-infra.git ~/eng-infra
```

### Step 2 — Go into the repo

```bash
cd ~/eng-infra
```

### Step 3 — Check out the branch with the new code

```bash
git fetch origin
git checkout MIT-3467-sonarqube
git pull
```

### Step 4 — Confirm you're logged in as `rohit.verma@takamul.ai`

```bash
az account show --query "{user:user.name,sub:name,tenant:tenantId}" -o json
```

Expected: `user = rohit.verma@takamul.ai`, `sub = cicd`, `tenant = af9e95b7-e99f-4ec1-9b02-429ffdc27b4e`.
If not, run `az login --tenant af9e95b7-e99f-4ec1-9b02-429ffdc27b4e` and pick subscription `[1] cicd`.

### Step 5 — Run the apply (only the 4 SAML resources; nothing else gets touched)

```bash
cd cicd/terraform/azure
terraform init -input=false
terraform apply -auto-approve \
  -target=azuread_application.sonarqube_saml \
  -target=azuread_service_principal.sonarqube_saml \
  -target=azuread_service_principal_token_signing_certificate.sonarqube_saml \
  -target=azuread_app_role_assignment.sonarqube_saml_devops_admin
```

### Step 6 — Verify the outputs

```bash
echo "=== application_id (NOT sensitive) ==="
terraform output -raw sonarqube_saml_application_id
echo
echo "=== signing cert PEM (SENSITIVE) ==="
terraform output -raw sonarqube_saml_signing_cert_pem
```

### Step 7 — Send me back

- The `application_id` value → paste in chat (it's a public identifier).
- The signing cert PEM → send via secure channel (1Password / signed Slack DM / encrypted email — not plain chat).

Done. You can `az logout` after if you want.

## What happens after Rohit sends the values

1. `application_id` → committed to `cicd/ansible/inventory/group_vars/cloud.yml` on branch `MIT-3467-sonarqube`, pushed.
2. Cert PEM → headers stripped, pushed to `mithril-shared-kv` as `cicd-sonarqube-saml-cert`. (`mohd.tahir` already has KV `Set` perm via the access policy grant on 2026-04-27.)
3. T02, T03, T04 marked complete.
4. Continue with T08 (helm deploy), T09 (Cloudflare apply), T10 (manual SAML smoke), T11 (e2e).
