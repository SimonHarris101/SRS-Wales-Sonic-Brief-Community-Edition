# SRS Redeployment Guide

Comprehensive checklist for deploying a new SRS instance of Sonic Brief into a **separate Azure subscription / tenant**.

> Goal: Achieve a clean, functional environment with minimal manual drift from IaC (Terraform) and correct Entra ID + resource identities.

---
## 1. Prerequisites
| Item | Notes |
|------|-------|
| Azure Subscription | Owner or Contributor + User Access Administrator to assign roles |
| Terraform Backend | (Recommended) Remote state (e.g., Azure Storage) per environment |
| Two Entra App Registrations | Backend API & Frontend SPA (single tenant or multi-tenant decision) |
| Azure OpenAI Quota | Region + model availability (gpt‑4o / embeddings) |
| Speech Service Quota | Region (latency + compliance) |
| Domain / Custom Hostnames (Optional) | DNS + certificate plan |

> **Security Note:** Never commit actual secrets, API keys, or production GUIDs to source control. Use placeholders in documentation and manage secrets through Azure App Settings or Key Vault.

## 2. Create / Configure Entra ID App Registrations

> **📖 Detailed Instructions:** For complete step-by-step guidance, see [`Entra-ID-App-Registration-Guide.md`](./Entra-ID-App-Registration-Guide.md)

| Step | Backend API App | Frontend SPA App |
|------|-----------------|------------------|
| Platform | Web/API (expose API) | SPA / Single Page Application |
| Expose API | Application ID URI (e.g., `api://<backend-client-id>`) | N/A |
| Scope | `user_impersonation` (add description, enabled) | Add as delegated permission (API Permissions) |
| Redirect URIs | Optional Web/SPA URIs for backend auth flows (see note) | Frontend URL(s): local dev + deployed SWA |
| Certificates & Secrets | Not required for implicit MSAL SPA | None |
| Token Config | (Optional) Add groups/roles claims if needed | N/A |

Record:
* Tenant ID
* Backend Client ID
* Frontend Client ID
* Audience / App ID URI
* Scope full string: `api://<backend-client-id>/user_impersonation`

> **Backend Redirect URIs (when using App Service Authentication):** In addition to the frontend SPA redirects, configure the backend API app registration with Web/SPA redirect URIs that match your backend and frontend hosts, including **both** the plain host and the paths `/auth/callback` and `/redirect` (for example: `https://<backend-app>.azurewebsites.net`, `https://<backend-app>.azurewebsites.net/auth/callback`, `https://<frontend-host>` and `https://<frontend-host>/redirect`). These are used by EasyAuth / OAuth flows to complete sign-in and redirect the user back to the SPA.

## 3. Prepare Terraform Variables
Create `terraform.tfvars` (or use `.auto.tfvars`) based on `infra/terraform..tfvars.sample`:

```hcl
# Required
subscription_id = "<new-sub-id>"

# Transcription model selection (matches variables.tf default)
transcription_model = "AZURE_AI_SPEECH"  # or "gpt-4o" if using GPT-4o audio

# Environment-specific overrides
allow_origins        = "https://your-static-web-app.azurestaticapps.net"
backend_log_level    = "INFO"
enable_swagger_oauth = false
```

> Start with `terraform..tfvars.sample` as your template and update only the values you need per environment (subscription ID, static web app origin, logging level, swagger OAuth toggle, and transcription model if different from the default).

## 4. Terraform Apply Order (if splitting state)
If using a single state file, a single `terraform apply` is sufficient. If modularizing:
1. Resource Group + Log Analytics
2. Networking / Storage / Cosmos
3. OpenAI + Speech
4. Function App + App Service (backend)
5. Static Web App

## 4.1 Pre-Terraform Backend Setup (Remote State)

If you use a remote state backend (recommended for shared/production environments), you must create the **Terraform state resource group, storage account, and container** *before* running `terraform init` in `infra/`.

The pattern used by the original deployment is:

1. **Login to Azure** (if not already logged in):

  ```powershell
  az login
  ```

2. **Create the resource group named `terraform`** (to hold the state storage account):

  ```powershell
  az group create --name terraform --location eastus
  ```

3. **Create the storage account and `tfstate` container**:

  For Windows PowerShell (uncomment and run):

  ```powershell
  $RANDOM_SUFFIX = [System.Guid]::NewGuid().ToString().Substring(0,4)

  # Create the storage account with random suffix
  $STORAGE_ACCOUNT = "terraform$RANDOM_SUFFIX"
  echo "Creating storage account: $STORAGE_ACCOUNT"
  az storage account create --name $STORAGE_ACCOUNT --resource-group terraform --sku Standard_LRS --encryption-services blob
  $ACCOUNT_KEY = az storage account keys list --resource-group terraform --account-name $STORAGE_ACCOUNT --query '[0].value' -o tsv

  # Create the container named 'tfstate'
  az storage container create --name tfstate --account-name $STORAGE_ACCOUNT --account-key $ACCOUNT_KEY

  echo "Storage account created: $STORAGE_ACCOUNT"
  ```

  For Linux/macOS shells (bash/zsh):

  ```bash
  RANDOM_SUFFIX=$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 4 | head -n 1)

  STORAGE_ACCOUNT="terraform${RANDOM_SUFFIX}"
  echo "Creating storage account: $STORAGE_ACCOUNT"
  az storage account create --name $STORAGE_ACCOUNT --resource-group terraform --sku Standard_LRS --encryption-services blob
  ACCOUNT_KEY=$(az storage account keys list --resource-group terraform --account-name $STORAGE_ACCOUNT --query '[0].value' -o tsv)

  az storage container create --name tfstate --account-name $STORAGE_ACCOUNT --account-key $ACCOUNT_KEY

  echo "Storage account created: $STORAGE_ACCOUNT"
  ```

4. **Configure the Terraform backend** in `infra/backend.tf` (copy from `backend.tf.sample` and replace placeholders):

  ```hcl
  terraform {
    backend "azurerm" {
     resource_group_name  = "terraform"              # RG holding the remote state storage account
     storage_account_name = "<your-terraform-storage-account-name>"  # e.g. terraformabcd (must exist)
     container_name       = "tfstate"                # Must exist within the storage account
     key                  = "<env>.tfstate"          # Unique key per environment (dev/test/prod)
     # use_azuread_auth   = true                      # Optional: use Azure AD auth instead of access keys
    }
  }
  ```

5. **Re-initialize Terraform** from the `infra/` folder so it picks up the remote backend:

  ```powershell
  cd infra
  terraform init -reconfigure
  ```

After this one-time setup, subsequent `terraform plan`/`terraform apply` commands will read and write state to the `terraform` RG’s storage account instead of using local state files.

## 5. Post-Provision Role Assignments
| Principal | Resource | Role | Why |
|----------|----------|------|-----|
| Backend App Service (MSI) | Cosmos DB Account | Cosmos DB Built-in Data Contributor | Read/write jobs, logs |
| Function App (MSI) | Storage Account | Storage Blob Data Contributor | Audio blob access |
| Function App (MSI) | Cosmos DB Account | Cosmos DB Built-in Data Contributor | Write transcription / analysis |
| (Optional) Backend MSI | Key Vault | Key Vault Secrets User | Secret retrieval if using Key Vault |

Validate via Portal or CLI (ARG queries) that assignments exist.

## 6. Configure Frontend
After first SWA deploy Azure assigns a hostname. Update:
* `frontend_static_hostname` variable (for subsequent applies)
* `allow_origins` in `infra/terraform.tfvars` to the exact SWA origin (no trailing slash)
* Ensure platform-level CORS (App Service “CORS” blade) does not contradict the app-level `ALLOW_ORIGINS` setting

## 7. Application Settings (Runtime Drift Watchlist)
Ensure the following Terraform-provisioned settings match portal runtime:
| Setting | Backend | Function App | Frontend (VITE_*) |
|---------|---------|--------------|-------------------|
| AUTH_METHOD | entra | N/A | VITE_AUTH_METHOD=entra |
| AZURE_* (tenant/client/audience) | ✔ | (subset if needed) | VITE_AAD_TENANT, VITE_AAD_CLIENT_ID |
| RETENTION / JOB vars | ✔ | ✔ (if functions read them) | VITE_RETENTION_DAYS (UI display only) |
| OPENAI_* | ✔ | (only if functions call analysis) | N/A |
| SPEECH_* | ✔ | ✔ | N/A |

## 8. Deployment Steps (End-to-End)
1. `terraform init && terraform apply` (confirm output endpoints)
2. Build & deploy backend (CI/CD or Terraform-driven `backend.zip`)
3. Package & deploy functions (Terraform-driven `az-func-audio.zip`)
4. Build frontend & deploy to SWA (`swa deploy` or GitHub Action)
5. Update SWA hostname variable & re-apply Terraform (so backend CORS stays consistent)
6. Smoke test: login (MSAL), call `/auth/me`, upload audio, observe transcription & summary, retention dry-run logs.

## 8.1. Manual Deployment Fallback (When Terraform Doesn't Deploy Code)

> **⚠️ Known Issue**: While Terraform builds Azure resources correctly, it sometimes fails to deploy application code to the assets. Use these manual commands as a fallback.

### Prerequisites for Manual Deployment

Before running manual deployment commands:

1. Ensure Terraform has successfully created all Azure resources
2. Note the resource names from Terraform output (they contain random suffixes)
3. Have deployment tokens and credentials ready

### Deploy Backend API

```bash
# From the repo root 
cd infra

# Ensure zip package exists (Terraform's archive_file will build backend.zip)
terraform init
terraform plan

# Deploy backend using modern OneDeploy (zip) API
az webapp deploy \
  --resource-group <resource-group-name> \
  --name <backend-app-service-name> \
  --type zip \
  --src-path backend.zip \
  --timeout 1200
```

**Example:**

```bash
az webapp deploy \
  --resource-group myorg-sb-v2t-dev-v2-accelerator-rg \
  --name myorg-sb-v2t-dev-v2-echo-brief-backend-api-xxxx \
  --type zip \
  --src-path backend.zip \
  --timeout 1200
```

> **Note:** `backend.zip` is generated by Terraform from the `backend_app` folder via the `archive_file` data source and is also used by Terraform’s `null_resource` to perform automated zip deploys during `terraform apply`. If `terraform plan` fails, fix plan errors before attempting manual deployment.

### Deploy Function App

```bash
# From the repo root 
cd infra

# Ensure az-func-audio.zip exists via Terraform
terraform init
terraform plan

# Deploy function app using modern OneDeploy (zip) API
az webapp deploy \
  --resource-group <resource-group-name> \
  --name <function-app-name> \
  --type zip \
  --src-path az-func-audio.zip \
  --timeout 1200
```

**Example:**

```bash
az webapp deploy \
  --resource-group myorg-sb-v2t-dev-v2-accelerator-rg \
  --name myorg-sb-v2t-dev-v2-audio-processor-xxxx \
  --type zip \
  --src-path az-func-audio.zip \
  --timeout 1200
```

> **Note:** Although this is an Azure Function App, the underlying platform is the same App Service and supports `az webapp deploy` for zip (OneDeploy) deployments. The zip content is produced from the `az-func-audio` folder via Terraform's `archive_file` data source and is also used by Terraform’s `null_resource` to perform automated zip deploys during `terraform apply`.

### Deploy Frontend (Static Web App)

```bash
# Navigate to frontend_app folder
cd frontend_app

# Install dependencies and build (using pnpm)
pnpm install
pnpm build

# Deploy to Static Web App using SWA CLI
swa deploy ./dist \
  --env=production \
  --deployment-token=<your-deployment-token>
```

> **📝 Note:** This project uses `pnpm` as the package manager. If you prefer `npm`, use these equivalent commands:
>
> ```bash
> npm install
> npm run build
> ```

**Example:**

```bash
swa deploy ./dist \
  --env=production \
  --deployment-token=abcd1234efgh5678ijkl9012mnop3456qrst7890uvwx1234yz567890ab
```

### Getting Required Values

**Resource Names:** Check Terraform output or Azure Portal for exact resource names with random suffixes.

**Deployment Token:**

1. Go to Azure Portal → Static Web Apps → Your SWA resource
2. Navigate to "Deployment tokens"
3. Copy the deployment token

**Subscription ID:**

```bash
az account show --query id --output tsv
```

**Resource Group:** Check Terraform variables or output for the exact resource group name.

### Manual Deployment Order

1. **Backend API** - Deploy first as other components may depend on it
2. **Function App** - Deploy audio processing functions
3. **Frontend** - Deploy last to ensure backend is ready

### Verification After Manual Deployment

After manual deployment, verify:

* Backend API responds at `https://<backend-name>.azurewebsites.net/health`
* Function App shows deployed functions in Azure Portal
* Frontend loads correctly at Static Web App URL
* End-to-end audio processing workflow works

### Troubleshooting Manual Deployment

**Backend deployment fails:**

* Ensure `backend.zip` exists in infra folder (run `terraform plan` to generate)
* Check App Service logs for deployment errors

**Function deployment fails:**

* Verify `az-func-audio.zip` exists in infra folder
* Try without `--build-remote true` flag if remote build fails
* Check Function App logs for errors

**Frontend deployment fails:**

* Verify SWA deployment token is correct and not expired
* Ensure `pnpm build` completed successfully
* Check if `./dist` folder contains built files

## 9. Likely Issues & Mitigations
| Category | Symptom | Cause | Mitigation |
|----------|---------|-------|-----------|
| Auth – 401 | MSAL obtains token but API 401 | Scope mismatch (frontend using wrong `azure_backend_scope`) | Align scope string; purge cached token; re-login |
| Auth – Silent Legacy Use | Legacy endpoints active | AUTH_METHOD left as `both` | Set `auth_method="entra"` and redeploy |
| CORS | Browser blocked | SWA host not in allowed origins | Add SWA hostname & re-apply / restart |
| OpenAI | 429 / model not found | Region lacks model quota | Pick supported region; request quota increase |
| Speech | 403 / region mismatch | Speech resource region differs from function config | Align `voice_location` & function env |
| Storage Name Conflict | Apply fails | Global name already taken | Adjust `prefix`/`environment` to new unique string |
| Cosmos Throughput Throttle | High latency | Insufficient RU/s | Scale provisioned throughput or enable autoscale |
| Retention Deleting Too Soon | Missing historical jobs | Wrong `job_retention_days` | Increase value; disable automatic retention temporarily |
| Function Cold Starts | Slow first transcription | Consumption plan & concurrency | Consider Premium plan or prewarming strategy |
| Missing MSI Role | Runtime 403 Cosmos / Storage | Role assignment lag or missing | Re-run role assignment; wait propagation (~5m) |

## 10. Validation Checklist (Pre Go-Live)
* [ ] Login succeeds (MSAL) and access token audience matches `azure_audience`.
* [ ] Audio upload -> transcription -> summary pipeline completes.
* [ ] Cosmos containers populated (jobs, job_activity_logs, audit_logs).
* [ ] Retention dry-run (if enabled) logs candidate deletions only.
* [ ] OpenAI summarization returns content (no 429/401).
* [ ] Structured logs visible in Log Analytics.
* [ ] No legacy auth endpoints reachable (if disabled).
* [ ] Frontend displays correct environment name & retention days.
* [ ] Confirm the appropriate Entra security/M365 group(s) are assigned to the frontend and backend Enterprise Applications (and that **Assignment required** is enabled if you are restricting access).

## 11. Base vs SRS Differentiators Summary

| Aspect | Base | SRS |
|--------|------|-----|
| Auth Default | (Legacy or hybrid) | Entra only |
| Retention Config | Basic / partial | Parameterized + dry-run & receipt cleanup |
| Infra Variables | Minimal | Extended (costing, retention, alerts) |
| Security Features | Basic | Production-hardened with comprehensive controls |

---

Maintain this guide with each architectural change to prevent knowledge drift.
