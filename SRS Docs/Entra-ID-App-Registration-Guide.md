# Entra ID App Registration Setup Guide (SRS)

## Overview

Sonic Brief SRS requires **three** Entra ID App Registrations to implement secure authentication and machine-to-machine ingestion:

- **Backend API App** - Exposes the API scope & app role(s)
- **Frontend SPA App** - Requests delegated (user) API permission
- **Ingestion (Machine) App** - Obtains client credential tokens bearing the required app role (e.g. `ingestion_role`) for privileged upload endpoints

> Canonical scope string: Use the Terraform/input variable `azure_backend_scope` everywhere (frontend, backend Swagger OAuth, docs). Its value determines whether the literal is `.../user_impersonation` or `.../access_as_user`. Treat `azure_backend_scope` as the single source of truth to avoid drift.

## Prerequisites

- Entra ID tenant with appropriate permissions
- Global Administrator or Application Administrator role
- Access to Azure Portal (portal.azure.com)

---

## Part 1: Backend API App Registration

### Step 1: Create the Backend App Registration

1. Navigate to **Azure Portal** > **Microsoft Entra ID** > **App registrations**
2. Click **+ New registration**
3. Configure the registration:
    - **Name**: `Sonic-Brief-Backend-API` (or environment-specific name)
    - **Supported account types**:
       - **Single tenant** (recommended): Accounts in this organizational directory only
       - **Multi-tenant** (if needed): Accounts in any organizational directory
    - **Redirect URI**: Leave blank for now; you will optionally add backend OAuth redirect URIs later if using App Service Authentication.
4. Click **Register**

### Step 2: Expose an API

1. Go to **Expose an API**
2. Click **+ Add a scope**
3. For **Application ID URI**:
   - Accept the default: `api://{client-id}`
   - Or customize: `api://sonic-brief-backend` (if preferred)
4. Configure the scope name:
   - If your `azure_backend_scope` uses `user_impersonation`, set scope name to `user_impersonation` (recommended for consistency with Terraform defaults)
   - If your environment already standardizes on `access_as_user`, set scope name accordingly
5. Click **Add scope**

### Step 3: Application roles

- Add an app role with the value matching your Terraform/backend env `SERVICE_PRINCIPAL_UPLOAD_ROLE`.
- Example role: Display name `Ingestion Role`, Value set to your `SERVICE_PRINCIPAL_UPLOAD_ROLE`, Allowed member type `Application`.

### Step 4: Record Backend App Details

Copy and securely store:

- **Application (client) ID**: `{backend-client-id}`
- **Directory (tenant) ID**: `{tenant-id}`
- **Application ID URI**: `api://{backend-client-id}` or custom URI
- **Scope**: value of `azure_backend_scope` (e.g., `api://{backend-client-id}/user_impersonation`)

### Step 5: (Optional) Configure Backend Redirect URIs for App Service Authentication

If you enable **Authentication / Authorization (EasyAuth)** on the backend App Service and use Entra ID for login flows through the backend, configure additional redirect URIs on the **backend API app registration**:

1. Go to the backend app registration > **Authentication**.
2. Under **Web** (or **Single-page application**, depending on your flow), add redirect URIs such as:
   - `https://<your-backend-app>.azurewebsites.net` (root backend URL)
   - `https://<your-backend-app>.azurewebsites.net/auth/callback`
   - `https://<your-frontend-host>`
   - `https://<your-frontend-host>/redirect`

These URIs correspond to:

- The public URL of the backend (`https://<your-backend-app>.azurewebsites.net`).
- The plain backend URL (no path) used by some flows as a reply URL.
- A backend callback path used by EasyAuth or OAuth middleware (`/auth/callback`).
- A frontend origin and redirect helper endpoint (for example, `/redirect`) hosted on the same origin as your SPA.

> The exact hostnames will differ per environment, but the **paths** `/auth/callback` and `/redirect` should be preserved when configuring redirect URIs for the backend login and logout flows.

---

## Part 2: Frontend SPA App Registration

### Step 1: Create the Frontend App Registration

1. In **App registrations**, click **+ New registration**
2. Configure the registration:
   - **Name**: `Sonic-Brief-Frontend-SPA` (or environment-specific name)
   - **Supported account types**: Same as backend (single/multi-tenant)
   - **Redirect URI**:
     - Platform: **Single-page application (SPA)**
     - URI: `http://localhost:5173` (for development)
3. Click **Register**

### Step 2: Configure Authentication (Frontend) (Complete after Sonic Brief deployment)

1. Go to **Authentication**
2. Under **Single-page application**, add production redirect URIs:
   - `https://{your-static-web-app}.azurestaticapps.net`
   - `https://{custom-domain}` (if using custom domain)
3. Under **Implicit grant and hybrid flows**:
   - [x] **Access tokens** (for SPA flows)
   - [x] **ID tokens** (for user identification)
4. Under **Advanced settings**:
   - **Allow public client flows**: No
   - **Live SDK support**: No

### Step 3: Configure API Permissions (Frontend)

1. Go to **API permissions**
2. Click **+ Add a permission**
3. Select **APIs my organization uses** tab
4. Find and select your backend app: `Sonic-Brief-Backend-API`
5. Select **Delegated permissions**
6. Check the delegated permission matching `azure_backend_scope` (commonly `user_impersonation`)
7. Click **Add permissions**
8. Keep the default `Microsoft Graph > User.Read` permission
9. Optional: Click **Grant admin consent**

---

## Part 3: Ingestion (Machine) App Registration

### Purpose

Provides a distinct service principal representing an automated ingestion or intermediate system (NOT the backend itself). It receives an access token via the client credentials flow; the backend authorizes it by presence of the configured app role value in the token's `roles` claim.

### Step 1: Create the Ingestion App Registration

1. **Azure Portal** > **Microsoft Entra ID** > **App registrations** > **+ New registration**
2. Name: `Sonic-Brief-MobileUploader` (environment suffix recommended)
3. Supported account types: Single tenant (same as backend)
4. Redirect URI: Leave blank
5. Register

### Step 2: Add a Client Secret (temporary until cert / managed identity)

1. In the ingestion app: **Certificates & secrets** > **Client secrets** > **+ New client secret**
2. Description: `ingestion-initial`
3. Expiry: Choose shortest practical (e.g., 6 or 12 months)
4. Record the Secret **Value** securely (not the secret ID). Store in Key Vault or pipeline secret store; never commit.

### Step 3: Assign Backend App Role

- Ensure the backend app role value matches `SERVICE_PRINCIPAL_UPLOAD_ROLE` from Terraform/backend env.

Portal limitations sometimes hide the "Add assignment" for application principals. Use Graph / CLI:

```bash
az rest --method POST \
   --url https://graph.microsoft.com/v1.0/servicePrincipals/<backend-sp-object-id>/appRoleAssignedTo \
   --body '{"principalId":"<ingestion-sp-object-id>","resourceId":"<backend-sp-object-id>","appRoleId":"<app-role-id>"}'
```

Where:

- `<backend-sp-object-id>`: Service principal object id of backend API
- `<ingestion-sp-object-id>`: Service principal object id of ingestion app
- `<app-role-id>`: GUID of the backend app role (value `ingestion_role`)

Verification:

```bash
az rest --method GET \
   --url https://graph.microsoft.com/v1.0/servicePrincipals/<ingestion-sp-object-id>/appRoleAssignments
```

Expect an entry referencing the backend display name and the role GUID.

#### Full Azure CLI Walkthrough (Automated Retrieval)

These commands (run in Bash or PowerShell with Az CLI logged in and appropriate Graph permissions such as `Application.Read.All` + `AppRoleAssignment.ReadWrite.All`) will discover the IDs and perform the assignment without manual GUID lookup.

Set variables (replace the friendly names):

```bash
BACKEND_APP_ID="<backend-application-client-id>"    # e.g. 71bea96c-... (Application (client) ID)
INGESTION_APP_ID="<ingestion-application-client-id>" # e.g. 8cf5fb04-...
ROLE_VALUE="ingestion_role"                              # Or your configured SERVICE_PRINCIPAL_UPLOAD_ROLE
```

Resolve object (service principal & application) IDs and role id:

```bash
BACKEND_SP_ID=$(az ad sp show --id "$BACKEND_APP_ID" --query id -o tsv)
INGESTION_SP_ID=$(az ad sp show --id "$INGESTION_APP_ID" --query id -o tsv)
ROLE_ID=$(az ad app show --id "$BACKEND_APP_ID" --query "appRoles[?value=='$ROLE_VALUE'].id | [0]" -o tsv)
echo "Backend SP: $BACKEND_SP_ID"; echo "Ingestion SP: $INGESTION_SP_ID"; echo "Role Id: $ROLE_ID"
```

Assign role (Graph POST):

```bash
az rest --method POST \
   --url https://graph.microsoft.com/v1.0/servicePrincipals/$BACKEND_SP_ID/appRoleAssignedTo \
   --body "{\"principalId\":\"$INGESTION_SP_ID\",\"resourceId\":\"$BACKEND_SP_ID\",\"appRoleId\":\"$ROLE_ID\"}" \
   --output json
```

Verify:

```bash
az rest --method GET --url https://graph.microsoft.com/v1.0/servicePrincipals/$INGESTION_SP_ID/appRoleAssignments \
   --query "value[?appRoleId=='$ROLE_ID']"
```

PowerShell variant (escaping adjusted):

```powershell
$BackendAppId = '<backend-application-client-id>'
$IngestionAppId = '<ingestion-application-client-id>'
$RoleValue = 'ingestion_role'

$BackendSpId = az ad sp show --id $BackendAppId --query id -o tsv
$IngestionSpId = az ad sp show --id $IngestionAppId --query id -o tsv
$RoleId = az ad app show --id $BackendAppId --query "appRoles[?value=='$RoleValue'].id | [0]" -o tsv

az rest --method POST `
   --url "https://graph.microsoft.com/v1.0/servicePrincipals/$BackendSpId/appRoleAssignedTo" `
   --body "{`"principalId`":`"$IngestionSpId`",`"resourceId`":`"$BackendSpId`",`"appRoleId`":`"$RoleId`"}"

az rest --method GET --url "https://graph.microsoft.com/v1.0/servicePrincipals/$IngestionSpId/appRoleAssignments" `
   --query "value[?appRoleId=='$RoleId']"
```

Error handling tips:

- If you see `Insufficient privileges to complete the operation`, ensure your signed-in identity has the necessary Graph application role permissions or use an elevated account.
- If `ROLE_ID` resolves empty, confirm the role value matches exactly (case-sensitive) the `value` field of the app role in the backend application registration.
- Propagation delays are rare; re-run verification after 30–60 seconds if assignment not immediately visible.

### Step 4: Test Token (Client Credentials)

Request token:

```http
POST https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
client_id=<ingestion-client-id>&client_secret=<secret>&scope=api://<backend-client-id>/.default&grant_type=client_credentials
```

Decode JWT; confirm:

- `aud` = `api://<backend-client-id>` (or custom Application ID URI)
- `roles` contains your configured role value (matches `SERVICE_PRINCIPAL_UPLOAD_ROLE`)

---

## Part 4: Terraform Integration

Update `terraform.tfvars`:

```hcl
azure_tenant_id                  = "{tenant-id}"
azure_client_id                  = "{backend-client-id}"
azure_frontend_client_id         = "{frontend-client-id}"
azure_audience                   = "api://{backend-client-id}"
azure_backend_scope              = "api://{backend-client-id}/user_impersonation" # or access_as_user if standardized
service_principal_upload_role    = "<your-role-value>" # align with backend & app role value
```

---

## Part 5: Assigning Users via Entra Groups

To control who can sign in to Sonic Brief, use Entra security or Microsoft 365 groups and Enterprise Application assignments rather than granting access to "All users" by default.

### Step 1: Create an Access Group

1. In **Microsoft Entra ID** > **Groups** > **New group**.
2. Group type: **Security** (or **Microsoft 365**, following your organization’s standards).
3. Name: a clear, environment-specific name such as `Sonic-Brief-Users-Prod`.
4. Add your initial users (or other groups) as members.

### Step 2: Assign the Group to the Enterprise Applications

For each of the **frontend SPA** and **backend API** apps:

1. Go to **Microsoft Entra ID** > **Enterprise applications**.
2. Open the Enterprise Application corresponding to your frontend or backend app registration.
3. Under **Users and groups**:
   - (Optional but recommended) Enable **Assignment required** so only assigned users/groups can sign in.
   - Click **+ Add user/group** and select the access group created in Step 1.
4. Save the assignment.

This ensures only members of the designated group(s) can authenticate to Sonic Brief, while the app registrations, scopes, and permissions remain reusable across environments.

---

## Notes on scope string consistency

- The backend Swagger OAuth initialization may reference `access_as_user` in code. To avoid drift, configure the app to use the value of `azure_backend_scope` as the single source of truth.
- Frontend `.env` should use `VITE_AZURE_BACKEND_SCOPE` derived from the same value.
