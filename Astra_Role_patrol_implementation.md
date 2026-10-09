# Role Patrol — Windows-to-Azure implementation guide

**Version:** 1.0 · **Prepared:** 8 October 2026 · **Design:** [Astra_Role_Patrol.md, version 1.3](Astra_Role_Patrol.md)

This guide builds a synthetic-data demonstration in the subscription supplied below. The primary UI is an Azure Static Web Apps browser application; Logic Apps provide protected APIs; Power BI is optional. The commands are **PowerShell on a Windows laptop**, not Bash or Cloud Shell. Account access, offer eligibility, quotas, authentication and connector behavior must pass the checks below.

The supplied [implementation support folder](Astra_implementation_support) contains actual scripts, rather than references to a repository you must find. Keep this document and that folder together. The scripts are a reference implementation to validate and extend, not a production-certified IAM application. Section 20 maps the reference implementation's limits to the full design; do not claim those limits have already been removed.

## 1. What you will build, and what “free trial” means

```text
Windows laptop: author/build/deploy only
       |
Browser -> Azure Static Web Apps UI + Entra sign-in
       -> OAuth-protected Logic Apps API -> scoped SQL procedures

Synthetic SQL HR/roles/tickets -> Data Factory -> immutable SQL assessment
Live fictional Entra groups -> read-only Logic App -> observed memberships
Assessment -> employee-impact ranking -> evidence -> owner decision/audit
SQL reporting views -> optional Power BI for licensed analysts
Catalog -> optional AI Search -> candidate -> human confirmation
```

Only these application services are used: **Azure Static Web Apps**, Azure Logic Apps **Consumption**, Azure SQL Database, Azure Data Factory, Entra ID, optional Azure AI Search and optional Power BI. ServiceNow is mocked through Logic Apps/SQL. There is no live HR, SailPoint or ServiceNow connection. No App Service, Functions, Storage account, Key Vault, API Management, Power Automate or Power Apps is added. Static Web Apps is used for static browser hosting only; it has no Functions/API backend. Resource groups, provider registrations, managed identities, API connections and cost-management controls are infrastructure/configuration of those services, not additional application platforms.

Azure free-account credit is time- and amount-limited; it does not make Logic Apps actions, SQL outside an eligible offer, Data Factory runs or paid Search tiers permanently free. In Azure Portal, open **Subscriptions → sub1 → Overview** and **Cost Management + Billing**. Record the offer, remaining credit, expiry and spending-limit status. Do not remove a spending limit or upgrade to pay-as-you-go merely to make a command succeed. [Microsoft free-account billing guidance](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/avoid-charges-free-account).

For this exercise:

| Component | Starting choice | Cost/availability check |
|---|---|---|
| SQL | Eligible Azure SQL free-offer serverless database; stop at free-limit exhaustion | Verify the offer is applied and exhaustion behavior is AutoPause. Otherwise choose a priced alternative yourself before continuing. |
| Static Web Apps | Free tier, one static browser app | Verify Free SKU availability; no API backend or managed authentication customization. |
| Logic Apps | Consumption, few manual runs; no frequent recurrence | Managed connector actions may consume credit; no longer used for static-page delivery. |
| Data Factory | Manual pipeline runs initially | No Mapping Data Flow cluster, SSIS runtime or continuous trigger. |
| Entra | Fictional unlicensed users and direct security groups | User/app creation and Graph permission consent require tenant authority separate from subscription access. |
| Search | Omit until the core works; Free SKU if available | Free-tier capacity/features differ from paid tiers. Semantic ranking is not required. |
| Power BI | Omit unless your analyst account already has permitted access | Azure trial credit does not automatically provide Power BI sharing licenses. |

If available to your billing role, add a budget in **Cost Management → Budgets**, using an amount you choose and email thresholds at 50%, 80% and 100%. A budget is a notification, not a hard stop. Check actual cost after each deployment stage. No fixed dollar estimate is asserted here because duration, region, offer and usage determine it.

**Validation:** you can identify the correct subscription, its current offer and the consequence of credit exhaustion. If you cannot, stop before provisioning billable resources.

## 2. Build order and working-demo milestones

Percentages are of your available build time, not Azure provisioning guarantees.

| Time | Stages | Exit |
|---|---|---|
| 0–10% | 3–7: tools, context, identities, Logic Apps HTML card shell and a synthetic review action | A browser opens a labelled card, sees the agreed evidence layout and records a `SIMULATED_DEMO_DECISION`. |
| 10–25% | 8–10: SQL, seed and managed-identity connections | Synthetic counts pass; the card reads a frozen snapshot and writes an audited simulated decision. |
| 25–40% | 11–12: Graph collection and ADF assessment | 72 observed memberships, 8 affected UK Finance employees, immutable published evidence. |
| 40–50% | 13–14: Static Web Apps authentication proof and enhanced UI | Browser Overview → evidence → authenticated owner review → saved decision works without Power BI. |
| 50–75% | 15–17: history, what-if, outcomes and mock facade | Extra capabilities do not break either working UI mode. |
| 75–80% | 18–19: optional Search/Power BI | Add only if earlier stages are stable. |
| 80–100% | 20–22: gap closure, tests, rehearsal, shutdown | Honest scope, repeatable demo and documented cleanup. |

Do not postpone Static Web Apps authentication until the end; start its feasibility spike once the Logic Apps card has demonstrated the flow. If the restricted browser/Logic Apps combination fails its feasibility check, retain the labelled Logic Apps simulated-card demo and continue independent SQL/evidence work. Do not describe a simulated card decision as an authenticated manager decision.

### 2.1 UI delivery path: fast card first, enhanced portal second

Build two browser UI modes against the same SQL evidence and case tables:

1. **Logic Apps HTML review card — first:** a Consumption Logic App returns an HTML/CSS evidence card for the fixed UK Finance/Jane scenario. A companion POST action validates one of the three allowed outcomes and records `SIMULATED_DEMO_DECISION` with the actual workflow identity, evidence version, case version, rationale and idempotency key. Show the banner **“Demo fallback — simulated reviewer decision”** on every view. Use a short-lived callback URL only during the presentation; do not commit, distribute or use it as an Entra redirect URI.
2. **Static Web Apps portal — second:** build the full Role Patrol overview, role-detail and review experience. It uses MSAL/PKCE and protected owner-bound Logic Apps APIs. Only this route can claim an authenticated manager/role-owner decision.

The Logic Apps card is the planned half-time safety net, not a hidden security shortcut. It must use the same stored-procedure validation and append-only audit path as the enhanced portal. It never grants access, changes role definitions or presents a simulated action as an IAM outcome.

## 3. Prepare the Windows laptop

1. Use a normal Windows user session. Open Edge or Chrome. Confirm you can access `https://portal.azure.com` and sign in to the intended tenant.
2. Open **Windows Terminal**. Install PowerShell 7 and Azure CLI if absent. These are local build tools, not deployed Azure services. If this is a managed laptop and installation is blocked, have the permitted software supplied; do not bypass endpoint policy.

```powershell
winget install --id Microsoft.PowerShell --exact --source winget
winget install --id Microsoft.AzureCLI --exact --source winget
winget install --id OpenJS.NodeJS.LTS --exact --source winget
```

3. Close Terminal, reopen it and choose **PowerShell 7**. `node`/`npm` are used only to bundle the browser authentication library on your laptop. No Node server is deployed or run for end users.
4. Run:

```powershell
$PSVersionTable.PSVersion
az version
node --version
npm.cmd --version
```

5. If `winget` is unavailable, install the same tools using their official installers: [PowerShell](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows), [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows), [Node.js](https://nodejs.org/en/download). Restart Terminal afterward.
6. Use PowerShell's `npm.cmd` explicitly; it avoids accidentally selecting a blocked `npm.ps1` shim. Run `Get-ExecutionPolicy -List` before invoking the supplied scripts. If a managed policy disables scripts, use your organization's approved execution process. On your own unmanaged laptop, if MachinePolicy and UserPolicy are Undefined, you can deliberately enable locally reviewed scripts with `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`; review the confirmation. Do not set machine-wide policy to Unrestricted. If a downloaded file is separately blocked, inspect it, then use `Unblock-File -LiteralPath <that-file>` only if permitted. Unblock-File does not override a Restricted execution policy.
7. Open your existing project folder:

```powershell
$PROJECT = 'C:\Users\arvin\OneDrive\Documents\MyProjects\Hackathon_2026'
Set-Location -LiteralPath $PROJECT
$SUPPORT = Join-Path $PROJECT 'Astra_implementation_support'
Get-ChildItem -LiteralPath $SUPPORT
$WORK = Join-Path $env:LOCALAPPDATA 'RolePatrolBuild'
New-Item -ItemType Directory -Path $WORK -Force | Out-Null
```

Use `$WORK` for deployment bodies, generated identity maps and callback configuration. This avoids automatically copying those artifacts into your OneDrive project folder. Do not store passwords or bearer tokens in files or transcripts.

8. Install the SQL command module under your own account:

```powershell
Install-Module SqlServer -Scope CurrentUser -Repository PSGallery
Import-Module SqlServer
Get-Command Invoke-Sqlcmd
```

Review any repository trust prompt rather than changing global trust settings. If this module cannot be installed, Azure SQL's portal Query Editor can run the SQL files one at a time, but token-based script execution below will not be available. Record that alternative rather than pretending the command ran.

**Validation:** PowerShell major version is at least 7, `az version`, `node --version`, `npm.cmd --version` and `Get-Command Invoke-Sqlcmd` succeed. All listed support files exist. No Azure resource has been created yet.

**Connect the dots:** the laptop prepares deployment files. The finished application runs from Azure and a browser; managers do not install these tools.

## 4. Pin and guard the exact subscription

Execute this block in your working PowerShell window. Keep the names for the entire build; do not generate a new suffix on every stage.

```powershell
$ErrorActionPreference = 'Stop'
$EXPECTED_SUBSCRIPTION_NAME = 'sub1'
$EXPECTED_SUBSCRIPTION_ID = 'a6b89415-776a-460e-8f87-802e172a7479'
$EXPECTED_TENANT_ID = 'cca2c4e5-a8a4-46a4-9354-abee5cc16a67'
$LOCATION = 'westus2'

function AzJson([string[]]$Arguments) {
    $output = & az @Arguments --output json
    if ($LASTEXITCODE -ne 0) { throw "Azure CLI failed: $($Arguments[0])" }
    if ($output) { ($output -join "`n") | ConvertFrom-Json }
}
function Assert-RpContext {
    $account = AzJson @('account','show')
    if ($account.id -ne $EXPECTED_SUBSCRIPTION_ID -or
        $account.tenantId -ne $EXPECTED_TENANT_ID -or
        $account.name -ne $EXPECTED_SUBSCRIPTION_NAME -or
        $account.state -ne 'Enabled') {
        throw 'Wrong or disabled subscription/tenant. No deployment is permitted.'
    }
}

az login --tenant $EXPECTED_TENANT_ID
if ($LASTEXITCODE -ne 0) { throw 'Sign-in failed.' }
az account set --subscription $EXPECTED_SUBSCRIPTION_ID
if ($LASTEXITCODE -ne 0) { throw 'Subscription selection failed.' }
Assert-RpContext
$Me = AzJson @('ad','signed-in-user','show')

$statePath = Join-Path $WORK 'names.json'
if (Test-Path -LiteralPath $statePath) {
    $Names = Get-Content -LiteralPath $statePath -Raw | ConvertFrom-Json
} else {
    $suffix = [guid]::NewGuid().ToString('N').Substring(0,8)
    $Names = [pscustomobject]@{
        Prefix = "rp$suffix"
        ResourceGroup = "rg-rolepatrol-$suffix"
        SqlServer = "sql-rp-$suffix"
        Database = 'rolepatrol'
        Factory = "adf-rp-$suffix"
        StaticWeb = "swa-rp-$suffix"
        Shell = "la-rp-shell-$suffix" # Legacy proof shell only; not an OAuth redirect host.
        ApiA = "la-rp-api-a-$suffix"
        ApiB = "la-rp-api-b-$suffix"
        Collector = "la-rp-collect-$suffix"
    }
    $Names | ConvertTo-Json | Set-Content -LiteralPath $statePath -Encoding utf8
}
if (-not ($Names.PSObject.Properties.Name -contains 'StaticWeb')) {
    $Names | Add-Member -NotePropertyName StaticWeb -NotePropertyValue ("swa-rp-" + $Names.Prefix.Substring(2))
    $Names | ConvertTo-Json | Set-Content -LiteralPath $statePath -Encoding utf8
}
$RG = $Names.ResourceGroup
$SQLSERVER = $Names.SqlServer
$DB = $Names.Database
$ADF = $Names.Factory
$ARM = "https://management.azure.com/subscriptions/$EXPECTED_SUBSCRIPTION_ID/resourceGroups/$RG"
$Names
```

When opening a new terminal, recreate the helper functions and expected constants and reload `names.json`; do not blindly repeat object-creation steps. `AzJson` checks external-command failures because PowerShell's `$ErrorActionPreference` alone does not reliably catch a nonzero exit code from `az`.

Check resource permissions in **Subscriptions → sub1 → Access control (IAM) → View my access**. You need permission to create the listed resources and their connections. Separately check Entra permissions to create demo users/groups, app registrations and grant Graph application permissions. Subscription Owner/Contributor does not automatically grant those tenant privileges.

Register only the required providers:

```powershell
Assert-RpContext
foreach ($provider in @('Microsoft.Logic','Microsoft.Web','Microsoft.Sql','Microsoft.DataFactory')) {
    az provider register --namespace $provider
    if ($LASTEXITCODE -ne 0) { throw "Provider registration failed: $provider" }
}
foreach ($provider in @('Microsoft.Logic','Microsoft.Web','Microsoft.Sql','Microsoft.DataFactory')) {
    az provider show --namespace $provider --query registrationState -o tsv
}
```

`Microsoft.Web` is required for managed **API connections** used by Consumption Logic Apps; this guide does not create an App Service. Wait until registration states are `Registered`. If policy denies registration, stop that branch and obtain an approved resource; do not silently change subscription or region.

Create only the dedicated group:

```powershell
Assert-RpContext
az group create --name $RG --location $LOCATION --tags Project=RolePatrol Data=Synthetic
if ($LASTEXITCODE -ne 0) { throw 'Resource-group creation failed.' }
az group show --name $RG --query '{name:name,location:location}' -o table
```

**Validation:** exact subscription name, subscription ID and tenant ID match; group location is `westus2`; required providers are registered. Save these non-secret results in your build notes.

## 5. Create fictional directory identities and record their real IDs

This stage is **setup administration**, not runtime remediation. It creates fictional lab users/groups; Role Patrol's runtime will only read them.

1. In Entra admin center, open **Identity → Overview / Custom domain names**. Identify a verified domain, normally an existing `*.onmicrosoft.com` domain in this tenant. Do not invent a domain or use `example.test` as an actual sign-in domain.
2. Verify directory-role permissions. If you lack user/group creation rights, an authorized tenant administrator must perform this stage and supply the maps.
3. Set the domain by entering the observed value:

```powershell
$VERIFIED_DOMAIN = Read-Host 'Enter an existing verified domain from the EXPECTED tenant'
$MAPDIR = Join-Path $WORK 'directory'
& (Join-Path $SUPPORT 'New-DemoDirectory.ps1') `
    -VerifiedDomain $VERIFIED_DOMAIN -Prefix $Names.Prefix -OutputDirectory $MAPDIR
```

4. The script prompts privately for different temporary passwords for Owner A (`E029`) and Owner B (`E030`). Remember them using your approved password handling method. Other accounts get random passwords that are not saved. The script saves IDs and UPNs, not passwords.

   For an approved agent-run setup, the script also accepts `-OwnerAPassword` and `-OwnerBPassword` as PowerShell `SecureString` values. Generate these in memory and store them only in a Windows-user-encrypted `.clixml` file under `$WORK`, never in this project, source control, browser configuration or a transcript. This permits a later local sign-in test without exposing the passwords in deployment output. The supplied script still forces a password change at first sign-in.
5. Open separate private browser sessions for the two owners. Sign in once, change the temporary password, and complete required security registration/MFA. Do not disable tenant security defaults to simplify the demo.
6. Load the maps:

```powershell
$Users = Import-Csv -LiteralPath (Join-Path $MAPDIR 'identity-map.csv')
$Groups = Import-Csv -LiteralPath (Join-Path $MAPDIR 'group-map.csv')
$OWNER_A_OID = ($Users | Where-Object EmployeeId -eq 'E029').ObjectId
$OWNER_B_OID = ($Users | Where-Object EmployeeId -eq 'E030').ObjectId
if ($Users.Count -ne 30 -or $Groups.Count -ne 8) { throw 'Directory setup incomplete.' }
$Users | Where-Object EmployeeId -in @('E029','E030') | Format-Table EmployeeId,UPN,ObjectId
```

7. In Entra **Groups**, inspect each prefixed group. Expected direct-member counts: Core 29, Reporting 20, Finance Input 20, Germany Reporting 1, Admin 1, Export 1, Operations 0, Legacy 0. Total 72. Empty groups are intentional.

If setup fails halfway, inspect the saved maps and tenant objects. Do not rerun with a fresh prefix and leave duplicate populations behind. The script refuses an existing user prefix. Repair the incomplete group/member setup explicitly through Entra using the same maps, then create `identity-map.sql` using the format shown in the existing output. Never delete unrelated tenant identities.

**Validation:** 30 fictional users, 8 direct-membership groups, 72 membership pairs; both owners can sign in; mapping files contain actual GUIDs. These files are setup instructions/evidence, not runtime observations.

**Connect the dots:** fictional HR employee `E021` and an Entra user have different IDs. The map is how a later Graph observation becomes Jane's observed entitlement.

## 6. Register the API and browser applications; create the Static Web Apps host

### 6.1 API registration

Use **Entra → App registrations → New registration**:

1. Name: `<prefix>-RolePatrol-API`; supported accounts: **this organizational directory only**; leave redirect URI blank.
2. Register. Copy **Application (client) ID**, not Object ID, into `$API_CLIENT_ID` below.
3. Open **Manifest**. In the Microsoft Graph-format manifest, set `api.requestedAccessTokenVersion` to `2`; preserve other fields. Save. This guide's audience/issuer checks require v2 access tokens.
4. Open **Expose an API → Add Application ID URI**. Use `api://<API client ID>`.
5. Add scope `RolePatrol.Access`, consent by admins, enabled. Use clear consent descriptions: read permitted Role Patrol evidence and submit permitted review requests. This scope does not grant directory writes.

### 6.2 Browser registration

1. Create `<prefix>-RolePatrol-Browser`, single tenant. Do not create a secret.
2. Copy its client ID into `$SPA_CLIENT_ID`.
3. Open **API permissions → Add a permission → My APIs → your Role Patrol API → Delegated permissions → RolePatrol.Access**. Add it.
4. Have the authorized administrator grant tenant consent. Remove unrelated requested permissions if they are not needed. Authentication uses OIDC; no Graph write permissions belong on this SPA.
5. The SPA redirect URI is added after the Static Web Apps hostname exists. Do not use `localhost`, a Power BI URL or a Logic Apps API callback as a substitute.

```powershell
$API_CLIENT_ID = Read-Host 'API Application (client) ID'
$SPA_CLIENT_ID = Read-Host 'Browser Application (client) ID'
$null = [guid]$API_CLIENT_ID
$null = [guid]$SPA_CLIENT_ID
@{ApiClientId=$API_CLIENT_ID;SpaClientId=$SPA_CLIENT_ID} |
    ConvertTo-Json | Set-Content -LiteralPath (Join-Path $WORK 'apps.json') -Encoding utf8
```

### 6.3 Create Azure Static Web Apps and register a non-secret redirect URI

Azure Static Web Apps is the approved browser host. It replaces the Logic Apps shell as the presentation/redirect host. The existing Logic Apps shell may remain as an unused non-sensitive proof artifact, but must not be registered as an OAuth redirect URI.

```powershell
Assert-RpContext
$STATIC_WEB = $Names.StaticWeb
az staticwebapp create --name $STATIC_WEB --resource-group $RG --location $LOCATION --sku Free `
    --tags Project=RolePatrol Data=Synthetic
if ($LASTEXITCODE -ne 0) { throw 'Static Web App creation failed.' }
$hostName = az staticwebapp show --name $STATIC_WEB --resource-group $RG --query defaultHostname -o tsv
if (-not $hostName) { throw 'Static Web App hostname was not returned.' }
$APP_URL = "https://$hostName"
$REDIRECT_URI = "$APP_URL/bridge.html"
$APP_ORIGIN = $APP_URL
[pscustomobject]@{StaticWeb=$STATIC_WEB;AppUrl=$APP_URL;RedirectUri=$REDIRECT_URI} |
    ConvertTo-Json | Set-Content -LiteralPath (Join-Path $WORK 'web-host.json') -Encoding utf8
```

In Entra **App registrations → `<prefix>-RolePatrol-Browser` → Authentication**, add platform **Single-page application** and paste the exact `$REDIRECT_URI`. Use organizational accounts only. The URI must be `https://<static-app-host>/bridge.html`; it must never include a Logic Apps callback, `sig`, `sp`, `sv`, deployment token or API URL. Keep it stable and use it in the browser configuration.

MSAL Browser v5 uses `bridge.html` as the same-origin popup return page. It must not receive a `Cross-Origin-Opener-Policy` header that severs popup communication. [Microsoft redirect bridge guide](https://learn.microsoft.com/en-us/entra/msal/javascript/browser/redirect-bridge), [redirect URI restrictions](https://learn.microsoft.com/en-us/entra/identity-platform/reply-url).

**Validation:** Static Web App resource reports `Free`; its `defaultHostname` is HTTPS; tenant/API/browser IDs are distinct and recorded; delegated scope and consent exist; exact SPA redirect URI is saved; no secret exists in browser configuration or an Entra redirect URI.

## 7. Prove browser OAuth and endpoint reachability before SQL spending

### 7.1 Deploy two principal-bound proof APIs

The reference pattern uses one API workflow per owner identity. It is intentionally small-cohort demo infrastructure. SQL will independently bind each workflow identity to a business principal; production would use a maintained shared authorization layer.

```powershell
$apiUrls = @{}
foreach ($entry in @(
    @{Name=$Names.ApiA;ObjectId=$OWNER_A_OID},
    @{Name=$Names.ApiB;ObjectId=$OWNER_B_OID}
)) {
    Assert-RpContext
    $file = Join-Path $WORK ($entry.Name + '.json')
    & (Join-Path $SUPPORT 'New-WorkflowBody.ps1') -Kind ApiProof -OutputFile $file `
        -TenantId $EXPECTED_TENANT_ID -ApiClientId $API_CLIENT_ID `
        -SpaClientId $SPA_CLIENT_ID -ReviewerObjectId $entry.ObjectId -AllowedOrigin $APP_ORIGIN
    az rest --method put --url "$ARM/providers/Microsoft.Logic/workflows/$($entry.Name)?api-version=2016-06-01" --body "@$file"
    if ($LASTEXITCODE -ne 0) { throw 'Proof API deployment failed.' }
    $cb = AzJson @('rest','--method','post','--url',"$ARM/providers/Microsoft.Logic/workflows/$($entry.Name)/triggers/manual/listCallbackUrl?api-version=2016-06-01")
    $uri = [UriBuilder]$cb.value
    $uri.Query = (($uri.Query.TrimStart('?').Split('&') | Where-Object { $_ -notmatch '^(sig|sp|sv)=' }) -join '&')
    $apiUrls[$entry.ObjectId] = $uri.Uri.AbsoluteUri
}
```

Open each workflow in Azure Portal → **Export template / JSON view** and verify:

- `sasAuthenticationPolicy.state = Disabled` on API triggers.
- Issuer is `https://login.microsoftonline.com/<expected tenant>/v2.0`.
- Audience is the API client ID; `tid` is the expected tenant; `oid` is the intended reviewer; `azp` is the SPA client ID; `scp` is `RolePatrol.Access`.
- The app registration issues v2 tokens; do not mix v1 issuer/audience values with this policy.
- A system-assigned managed identity exists on each API workflow. No SQL connection exists yet.

Microsoft documents the Consumption trigger authorization policy and separate SAS-disable setting. Merely adding OAuth does not remove the alternate SAS path. [Logic Apps security configuration](https://learn.microsoft.com/en-us/azure/logic-apps/set-up-security-permissions).

### 7.2 Build browser assets on Windows

```powershell
$WEB = Join-Path $SUPPORT 'web'
Push-Location $WEB
npm.cmd install --save-exact @azure/msal-browser@5
if ($LASTEXITCODE -ne 0) { throw 'MSAL installation failed.' }
npm.cmd install --save-dev --save-exact esbuild
if ($LASTEXITCODE -ne 0) { throw 'Bundler installation failed.' }
npm.cmd ls --depth=0
npm.cmd run build:app
if ($LASTEXITCODE -ne 0) { throw 'Application bundle failed.' }
npm.cmd run build:bridge
if ($LASTEXITCODE -ne 0) { throw 'Bridge bundle failed.' }
Pop-Location

$configFile = Join-Path $WORK 'browser-config.json'
@{tenantId=$EXPECTED_TENANT_ID;clientId=$SPA_CLIENT_ID;apiClientId=$API_CLIENT_ID;
  redirectUri=$REDIRECT_URI;apiByObjectId=$apiUrls} |
    ConvertTo-Json -Depth 10 | Set-Content -LiteralPath $configFile -Encoding utf8
& (Join-Path $WEB 'Build-Html.ps1') -ConfigFile $configFile

# The deployment token is a secret. SWA CLI stores interactive credentials in the OS keychain;
# do not put a deployment token in source, browser configuration, transcripts or OneDrive.
Push-Location $WEB
npm.cmd install --save-dev --save-exact @azure/static-web-apps-cli
npx.cmd swa login --subscription-id $EXPECTED_SUBSCRIPTION_ID --resource-group $RG `
    --tenant-id $EXPECTED_TENANT_ID --app-name $STATIC_WEB --use-keychain
npx.cmd swa deploy ./dist --env production
Pop-Location
```

The first package installation resolves a concrete v5 version and saves it in `package-lock.json`; retain that file with the build. Subsequent builds use `npm.cmd ci`, not another unrestricted update. Downloads occur on the developer laptop, not in the end-user browser. No package CDN is required at runtime.

Open `$APP_URL`, confirm that `index.html` and `bridge.html` are served from the Static Web Apps hostname, then click **Sign in**, choose Owner A and complete MFA. Open browser Developer Tools → Network. The API request should return the proof message. The application can still show “No completed assessment yet”; this stage proves authentication, not business data. Repeat with Owner B in a separate private profile.

### 7.3 CORS and same-origin gate — do not skip

Inspect the scheme/host/port of `$APP_URL` and each API URL. Different workflows in the same region are not guaranteed to receive the same origin. If origins differ, the browser sends an OPTIONS preflight before the bearer-token POST. Inspect that request in Developer Tools:

1. It must succeed without an OAuth token and return a policy allowing the exact app origin, POST and the Authorization/Content-Type headers.
2. The actual POST must still require OAuth and reject unauthorized callers.
3. A response header on the POST alone does not prove preflight works.

The supplied API proof sets an exact-origin response header, but does **not** claim to implement a platform-level unauthenticated preflight handler. Validate the actual Consumption endpoint behavior. If the platform handles it, continue; if OPTIONS is rejected or the browser blocks the exchange, this design's browser/API path is **not yet viable in that account**. Stop the web branch before building the larger application. Do not disable OAuth, publish a business API SAS, use `no-cors`, install a proxy service outside the allowlist or assume Postman success proves browser success. The original design made this an explicit feasibility gate.

This is a material limitation, not a hidden step: Logic Apps is a workflow host, and a universally working web-hosting/authentication arrangement cannot be guaranteed from the service allowlist alone. Keep the SQL/evidence branch useful while you resolve the allowed endpoint configuration or report the blocked requirement.

### 7.4 Negative tests

```powershell
# Expected: non-success HTTP status; no business action.
$anonymousAccepted = $false
try {
    Invoke-RestMethod -Method Post -Uri $apiUrls[$OWNER_A_OID] `
        -ContentType 'application/json' -Body '{"op":"overview"}'
    $anonymousAccepted = $true
} catch {
    Write-Host 'Inspect response: anonymous access must be rejected by the trigger.'
}
if ($anonymousAccepted) { throw 'Unexpected anonymous success: stop and correct trigger authorization.' }
```

Also test an A token against B's endpoint and vice versa using a temporary local browser configuration that deliberately routes to the other endpoint; restore the mapping afterward. Both must fail. Do not paste tokens into websites or transcripts. Verify no successful protected workflow run occurs for a wrong-owner token.

**Validation:** both owners can sign in and call only their own protected proof endpoint; anonymous and cross-owner calls fail; browser preflight works where needed. Record PASS/FAIL separately. A failed result is not a reason to loosen the policy.

**Connect the dots:** the browser receives a token for Role Patrol, Logic Apps validates it, and only the matching owner's backend executes. Knowing the application URL is insufficient to read evidence.

## 8. Create Azure SQL with Entra-only authentication

First confirm West US 2 supports the offered database configuration for your subscription. An unavailable SKU/offer must produce a deliberate decision, not an automatic paid upgrade.

```powershell
Assert-RpContext
az sql server create --resource-group $RG --name $SQLSERVER --location $LOCATION `
    --enable-ad-only-auth --external-admin-principal-type User `
    --external-admin-name $Me.userPrincipalName --external-admin-sid $Me.id
if ($LASTEXITCODE -ne 0) { throw 'SQL server creation failed.' }

az sql db create --resource-group $RG --server $SQLSERVER --name $DB `
    --edition GeneralPurpose --family Gen5 --capacity 2 --compute-model Serverless `
    --use-free-limit --free-limit-exhaustion-behavior AutoPause
if ($LASTEXITCODE -ne 0) { throw 'SQL free-offer creation failed. Review eligibility; do not auto-switch to paid.' }
az sql db show --resource-group $RG --server $SQLSERVER --name $DB -o json
```

If the installed CLI lacks these switches, update it through the approved installation path or use **Azure Portal → SQL databases → Create → Apply free offer**. Choose **Auto-pause until next month** when the free limit is reached. Do not select **Continue using for additional charges** unintentionally. Verify the resulting database's offer and cost settings. [SQL free offer](https://learn.microsoft.com/en-us/azure/azure-sql/database/free-offer?view=azuresql), [CLI free-offer parameters](https://learn.microsoft.com/en-us/cli/azure/sql/db?view=azure-cli-latest), [Entra-only server creation](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-only-authentication-create-server?view=azuresql).

### 8.1 Permit only the required network paths

1. Open the **SQL logical server → Networking**. Use the account's permitted public-endpoint configuration for this free-trial exercise. If policy mandates private endpoints and you cannot provide an approved path with the listed services, stop; do not broaden networking to bypass it.
2. Add **your current laptop IPv4** using **Add your client IPv4 address**. Save. Do not enter your private `192.168.*` Wi-Fi address.
3. Leave **Allow Azure services and resources to access this server** off. That broad rule is not limited to your subscription.
4. Later add the exact required regional outbound IPv4 ranges for the SQL managed connector and the selected West US 2 ADF integration runtime. Use current Microsoft-published addresses, not copied historic ranges. Azure service tags describe the service ranges; SQL firewall entries still need start/end addresses. [Logic Apps IP configuration](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config#outbound-ip-addresses), [ADF network guidance](https://learn.microsoft.com/en-us/azure/data-factory/azure-integration-runtime-ip-addresses).
5. Record each rule's purpose. These are shared service ranges, not an identity boundary; Entra/SQL grants remain mandatory. If a required range list exceeds your permitted firewall configuration, resolve the network design rather than enabling all Azure traffic.

For a single observed, approved IPv4, the CLI shape is:

```powershell
$LAPTOP_IPV4 = Read-Host 'Enter the public client IPv4 shown by SQL Networking'
$parsed = [System.Net.IPAddress]::Parse($LAPTOP_IPV4)
if ($parsed.AddressFamily -ne [System.Net.Sockets.AddressFamily]::InterNetwork) { throw 'An IPv4 is required.' }
az sql server firewall-rule create --resource-group $RG --server $SQLSERVER `
    --name RolePatrolLaptop --start-ip-address $LAPTOP_IPV4 --end-ip-address $LAPTOP_IPV4
if ($LASTEXITCODE -ne 0) { throw 'Firewall rule failed.' }
```

### 8.2 Connect using your Entra SQL administrator

```powershell
function Invoke-RpSqlFile([string]$File) {
    Assert-RpContext
    $token = az account get-access-token --resource https://database.windows.net/ --query accessToken -o tsv
    if ($LASTEXITCODE -ne 0) { throw 'SQL token acquisition failed.' }
    try {
        Invoke-Sqlcmd -ServerInstance "$SQLSERVER.database.windows.net" -Database $DB `
            -AccessToken $token -InputFile $File -AbortOnError -ErrorAction Stop
    } finally { $token = $null }
}
function Invoke-RpSql([string]$Query) {
    Assert-RpContext
    $token = az account get-access-token --resource https://database.windows.net/ --query accessToken -o tsv
    if ($LASTEXITCODE -ne 0) { throw 'SQL token acquisition failed.' }
    try {
        Invoke-Sqlcmd -ServerInstance "$SQLSERVER.database.windows.net" -Database $DB `
            -AccessToken $token -Query $Query -AbortOnError -ErrorAction Stop
    } finally { $token = $null }
}
Invoke-RpSql 'SELECT DB_NAME() AS DatabaseName, USER_NAME() AS DatabaseUser;'
```

If the database is paused, the first connection can fail while it resumes. Wait briefly, check status and retry. Do not treat an authentication or firewall failure as merely a pause. Never use `TrustServerCertificate` or disable encryption to suppress an unrelated error.

**Validation:** database is `rolepatrol`, caller is your configured Entra administrator, the free-offer limit behavior is explicit, and no SQL password is required.

## 9. Load schema, synthetic data and engine

Run these in order, stopping on the first error:

```powershell
Invoke-RpSqlFile (Join-Path $SUPPORT '01_schema_seed.sql')
Invoke-RpSqlFile (Join-Path $MAPDIR 'identity-map.sql')
Invoke-RpSqlFile (Join-Path $SUPPORT '02_engine.sql')
Invoke-RpSqlFile (Join-Path $SUPPORT '04_operations.sql')
Invoke-RpSql 'EXEC core.usp_Ingest;'
```

The schema/seed script refuses to run if its employee table already exists. It does not erase earlier work. Engine/operations procedures use `CREATE OR ALTER` and can be reapplied after review. If the initial schema fails halfway, inspect the error and deployed objects; do not rerun blindly against a partially created schema. For an unused demo database, explicitly create a fresh database and start the stage there; never drop a database containing required audit evidence.

The reference implementation uses the synthetic SQL `src` tables as the mock source. `core.usp_Ingest` validates them; `core.usp_Assess` freezes a transaction-consistent source batch into `Assessment.SourceJson` and typed evidence tables. This is a concrete compact alternative to a separate table-by-table `stg` copy in the HLD. Source and published evidence remain separate; production connectors would populate source/staging contracts before assessment.

Validate:

```powershell
Invoke-RpSql @'
SELECT 'Employees' Entity,COUNT(*) RowCount FROM src.Employee
UNION ALL SELECT 'Roles',COUNT(*) FROM src.Role
UNION ALL SELECT 'RoleVersions',COUNT(*) FROM src.RoleVersion
UNION ALL SELECT 'Entitlements',COUNT(*) FROM src.Entitlement
UNION ALL SELECT 'Moves',COUNT(*) FROM src.Move
UNION ALL SELECT 'Tickets',COUNT(*) FROM src.Ticket
UNION ALL SELECT 'IdentityMap',COUNT(*) FROM src.IdentityMap
UNION ALL SELECT 'GroupMap',COUNT(*) FROM src.GroupMap
UNION ALL SELECT 'ExpectedMemberships',COUNT(*) FROM src.ExpectedMember
UNION ALL SELECT 'ObservedMemberships',COUNT(*) FROM core.Member;
'@
```

Expected: **30, 6, 7, 8, 23, 40, 30, 8, 72, 0**, respectively. The zero observed membership count is correct: directory observations have not been collected yet.

**Connect the dots:** the 72 setup pairs say what the demo directory should contain. They are deliberately not inserted into the observation table; the next stage must read Entra to establish what it actually contains.

## 10. Connect service identities to SQL

### 10.1 Create Data Factory and the collector

1. In Azure Portal, open the resource group from Stage 4. Select **Create > Data Factory**. Use the saved factory name, `westus2`, version V2. Select **Configure Git later**. Create it and open **Identity**; confirm system-assigned identity is enabled.
2. In the same group create **Logic App (Consumption)** with the saved collector name. Do not select Standard: it adds hosting/storage requirements. Open **Identity > System assigned > On > Save**.
3. In its designer select **When an HTTP request is received**, method POST, schema `{"type":"object"}`. Add a temporary Response, status 200, body `{"message":"collector setup"}`. Save. Stage 11 replaces this response.
4. Read identity **principal IDs**, not application IDs:

```powershell
$ADF_OID = (AzJson @('resource','show','-g',$RG,'-n',$ADF,'--resource-type','Microsoft.DataFactory/factories')).identity.principalId
$COLLECTOR_OID = (AzJson @('resource','show','-g',$RG,'-n',$names.Collector,'--resource-type','Microsoft.Logic/workflows')).identity.principalId
$API_A_MI = (AzJson @('resource','show','-g',$RG,'-n',$names.ApiA,'--resource-type','Microsoft.Logic/workflows')).identity.principalId
$API_B_MI = (AzJson @('resource','show','-g',$RG,'-n',$names.ApiB,'--resource-type','Microsoft.Logic/workflows')).identity.principalId
Invoke-RpSqlFile (Join-Path $SUPPORT '05_status.sql')
$principalMap = @{rp_adf=$ADF_OID;rp_collect=$COLLECTOR_OID;rp_api_a=$API_A_MI;rp_api_b=$API_B_MI}
foreach ($entry in $principalMap.GetEnumerator()) {
    $id = [guid]$entry.Value
    if ($id -eq [guid]::Empty) { throw 'Missing managed identity' }
    $sid = '0x' + (($id.ToByteArray() | ForEach-Object { $_.ToString('X2') }) -join '')
    Invoke-RpSql "IF DATABASE_PRINCIPAL_ID('$($entry.Key)') IS NULL CREATE USER [$($entry.Key)] WITH SID=$sid, TYPE=E;"
}
Invoke-RpSql @'
GRANT EXECUTE ON OBJECT::core.usp_Ingest TO rp_adf;
GRANT EXECUTE ON OBJECT::core.usp_Assess TO rp_adf;
GRANT EXECUTE ON OBJECT::core.usp_CollectionStatus TO rp_adf;
GRANT EXECUTE ON OBJECT::core.usp_RunStatus TO rp_adf;
GRANT EXECUTE ON OBJECT::core.usp_Collect TO rp_collect;
GRANT EXECUTE ON OBJECT::core.usp_GroupMap TO rp_collect;
GRANT EXECUTE ON OBJECT::api.usp_Gateway TO rp_api_a;
GRANT EXECUTE ON OBJECT::api.usp_Gateway TO rp_api_b;
INSERT core.Principal(DbUser,EmployeeId)
SELECT 'rp_api_a','E029' WHERE NOT EXISTS(SELECT 1 FROM core.Principal WHERE DbUser='rp_api_a');
INSERT core.Principal(DbUser,EmployeeId)
SELECT 'rp_api_b','E030' WHERE NOT EXISTS(SELECT 1 FROM core.Principal WHERE DbUser='rp_api_b');
'@
```

SID/TYPE creation avoids requiring SQL to resolve names through Graph. Verify IDs against each resource's Identity page. [SQL managed-identity guidance](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?view=azuresql).

### 10.2 Permit connector network access

The laptop SQL firewall rule does not permit cloud connectors. Do not enable access from every Azure service to fix this.

1. Retrieve current regional service tags:

```powershell
$tags = AzJson @('network','list-service-tags','--location',$LOCATION)
$requiredTags = @($tags.values | Where-Object {
    $_.name -match '^(AzureConnectors|DataFactory)\.' -and $_.properties.region -eq $LOCATION
})
$requiredTags | Select-Object name,@{n='Prefixes';e={$_.properties.addressPrefixes -join ', '}}
if ($requiredTags.Count -lt 2) { throw 'Locate actual regional connector and Data Factory service tags before proceeding' }
```

2. Confirm the SQL managed connection and ADF Azure integration runtime actually use `westus2`. In ADF, create an Azure integration runtime with that explicit region instead of an auto-resolved runtime if necessary.
3. Convert each required IPv4 CIDR to a firewall range:

```powershell
function Convert-CidrRange([string]$Cidr) {
    $parts = $Cidr.Split('/')
    $bytes = [System.Net.IPAddress]::Parse($parts[0]).GetAddressBytes()
    if ($bytes.Length -ne 4) { return }
    $bits = [int]$parts[1]
    if ($bits -lt 0 -or $bits -gt 32) { throw 'Invalid prefix' }
    [uint64]$value = ([uint64]$bytes[0]*16777216)+([uint64]$bytes[1]*65536)+([uint64]$bytes[2]*256)+$bytes[3]
    [uint64]$size = [math]::Pow(2,32-$bits)
    [uint64]$start = [math]::Floor($value/$size)*$size
    [uint64]$end = $start+$size-1
    function Format-IPv4([uint64]$v) { '{0}.{1}.{2}.{3}' -f (($v -shr 24)-band 255),(($v -shr 16)-band 255),(($v -shr 8)-band 255),($v-band 255) }
    [pscustomobject]@{Start=(Format-IPv4 $start);End=(Format-IPv4 $end)}
}
$i=0
foreach ($prefix in ($requiredTags.properties.addressPrefixes | Sort-Object -Unique)) {
    $range=Convert-CidrRange $prefix
    if ($null -eq $range) { continue }
    $i++
    AzJson @('sql','server','firewall-rule','create','-g',$RG,'-s',$SQLSERVER,'-n',"rp-connectors-$i",'--start-ip-address',$range.Start,'--end-ip-address',$range.End) | Out-Null
}
```

These ranges are shared regional infrastructure, not private networking. SQL authentication still applies. If lab policy forbids public SQL or these ranges, this topology is blocked pending an approved network design. Do not silently add private endpoints or other services. Recheck ranges before a later event. [Logic Apps networking addresses](https://learn.microsoft.com/en-us/azure/logic-apps/logic-apps-limits-and-config#inbound-and-outbound-ip-addresses).

### 10.3 Create managed-identity SQL connections

For **each** API workflow and the collector:

1. Open Designer; add managed **SQL Server > Execute stored procedure (V2)**.
2. Create a **separate connection for that workflow**, using **Logic Apps Managed Identity**, system assigned. Do not reuse API A's connection for API B.
3. Server: `<saved-server>.database.windows.net`; database: `rolepatrol`. Collector procedure: `core.usp_GroupMap`. API procedure: `api.usp_Gateway`. Select custom values if necessary; never grant db_owner just to populate designer dropdowns.
4. Save the connection. Stages 11/13 complete these temporary actions.
5. In ADF Studio > Manage > Linked services > New > Azure SQL Database, name `ls_rolepatrol_sql`; choose the server/database, regional integration runtime and **System Assigned Managed Identity**. Test connection and save.

**Validate:** ADF connection test succeeds. SQL has four external users with named procedure grants. As setup administrator, test `EXECUTE AS USER='rp_api_a'; EXEC api.usp_Gateway N'{"op":"overview"}'; REVERT;`. Direct SELECT from `src.Employee` under that user must fail. After a failing impersonation test, run `REVERT` or close that connection. This tests SQL permissions, not browser sign-in.

**Connect the dots:** Entra authenticates the person to an owner-bound workflow; its SQL identity determines the audit actor. Client JSON cannot nominate the reviewer. The two-owner pattern is deliberately bounded; scaling limitations are in Stage 20.

## 11. Collect observed memberships from Entra

### 11.1 Grant Graph read permission

A tenant administrator must assign the collector managed identity the **application** permission `GroupMember.Read.All`. Subscription ownership alone is insufficient. Do not grant directory write permissions or grant this permission to the SPA/API workflows.

```powershell
$graphSp = @(AzJson @('ad','sp','list','--filter',"appId eq '00000003-0000-0000-c000-000000000000'"))[0]
$graphRole = @($graphSp.appRoles | Where-Object { $_.value -eq 'GroupMember.Read.All' -and $_.allowedMemberTypes -contains 'Application' })
if ($graphRole.Count -ne 1) { throw 'Graph permission not found uniquely' }
$grantFile = Join-Path $WORK 'graph-grant.json'
@{principalId=$COLLECTOR_OID;resourceId=$graphSp.id;appRoleId=$graphRole[0].id} | ConvertTo-Json | Set-Content -LiteralPath $grantFile -Encoding utf8
$existing = AzJson @('rest','--method','GET','--url',"https://graph.microsoft.com/v1.0/servicePrincipals/$COLLECTOR_OID/appRoleAssignments")
if (-not @($existing.value | Where-Object { $_.resourceId -eq $graphSp.id -and $_.appRoleId -eq $graphRole[0].id }).Count) {
    AzJson @('rest','--method','POST','--url',"https://graph.microsoft.com/v1.0/servicePrincipals/$COLLECTOR_OID/appRoleAssignments",'--body',"@$grantFile")
}
```

This permission is tenant-wide; the workflow's eight-group allowlist is an application constraint, not permission-level isolation. The fixture uses direct user members, not nested-group effective access. [Graph list-members documentation](https://learn.microsoft.com/en-us/graph/api/group-list-members?view=graph-rest-1.0).

### 11.2 Build the collector workflow

Rename actions before entering expressions. Designer Expression tab omits leading `@`; workflow JSON includes it.

1. Keep POST Request. Initialize String variables `CollectionId` = `guid()` and `NextLink` = empty string at workflow root.
2. SQL action `Start_collection`: `core.usp_Collect`, Action=`Start`, CollectionId=`variables('CollectionId')`; leave optional parameters at defaults.
3. Replace temporary Response with `Accepted`: status **202**, body `{"collectionId":"@{variables('CollectionId')}"}`, no Location header. Work continues after acknowledgment.
4. Add Scope `Collect`. Inside it execute `core.usp_GroupMap`, action `Read_groups`.
5. Add `For_each` over `body('Read_groups')?['ResultSets']?['Table1']`. Enable concurrency control, degree **1**. Check actual result-set casing in run output and adjust consistently if needed.
6. Inside loop set NextLink to `concat('https://graph.microsoft.com/v1.0/groups/',items('For_each')?['ObjectId'],'/members?$select=id')`.
7. Add Until: `equals(variables('NextLink'),'')`, limit 100 iterations, timeout `PT5M`. Its actions, in sequence:
   - Require `startsWith(variables('NextLink'),'https://graph.microsoft.com/')`; fail on another host.
   - HTTP `Read_page`, GET NextLink, managed identity authentication, audience `https://graph.microsoft.com/`. Use exponential retry for transient failures.
   - SQL `Save_page`: `core.usp_Collect`; Action=`Page`; CollectionId variable; EntitlementId=`items('For_each')?['EntitlementId']`; PageJson=`string(body('Read_page'))`; LastPage=`empty(body('Read_page')?['@odata.nextLink'])`.
   - Set NextLink=`coalesce(body('Read_page')?['@odata.nextLink'],'')`.
8. After the group loop, inside Scope, execute `core.usp_Collect`, Action=`Finish`, CollectionId variable.
9. Outside Scope add SQL `Mark_failed`, same procedure, Action=`Fail`. Configure Run after on Collect for **failed/timed out only**, then Terminate Failed. No second HTTP response is needed.
10. Save. After troubleshooting, secure Graph/SQL action inputs and outputs in Settings. Do not substitute seeded expected memberships if Graph fails.

### 11.3 Restrict invocation to ADF

Retain the workflow's definition/connections/identity and apply the same trigger authorization structure used in Stage 7, with a single managed-identity policy:

| Claim | Value |
|---|---|
| iss | `https://sts.windows.net/cca2c4e5-a8a4-46a4-9354-abee5cc16a67/` |
| aud | `https://management.azure.com/` |
| tid | supplied tenant ID |
| oid | actual `$ADF_OID` |

Omit the SPA user azp/scp checks for this machine identity. Disable SAS authentication. The ADF Web activity will request a token for the ARM resource. If token validation fails, inspect protected diagnostics and match the issuer/audience of the actual resource token; do not remove caller or audience validation. [ADF managed-identity authentication](https://learn.microsoft.com/en-us/azure/data-factory/control-flow-web-activity), [Logic Apps authorization policy](https://learn.microsoft.com/en-us/azure/logic-apps/set-up-security-permissions).

Apply the policy without replacing the collector definition:

```powershell
$collectorArm = "$ARM/providers/Microsoft.Logic/workflows/$($names.Collector)?api-version=2016-06-01"
$collectorResource = AzJson @('rest','--method','GET','--url',$collectorArm)
$collectorPolicy = @{triggers=@{
    sasAuthenticationPolicy=@{state='Disabled'}
    openAuthenticationPolicies=@{policies=@{adf=@{type='AAD';claims=@(
        @{name='iss';value="https://sts.windows.net/$EXPECTED_TENANT_ID/"},
        @{name='aud';value='https://management.azure.com/'},
        @{name='tid';value=$EXPECTED_TENANT_ID},
        @{name='oid';value=$ADF_OID}
    )}}}
}}
$collectorUpdate = @{
    location=$LOCATION;identity=@{type='SystemAssigned'}
    properties=@{
        state='Enabled';definition=$collectorResource.properties.definition
        parameters=$collectorResource.properties.parameters;accessControl=$collectorPolicy
    }
}
$collectorPolicyFile=Join-Path $WORK 'collector-policy.json'
$collectorUpdate | ConvertTo-Json -Depth 100 | Set-Content -LiteralPath $collectorPolicyFile -Encoding utf8
AzJson @('rest','--method','PUT','--url',$collectorArm,'--body',"@$collectorPolicyFile") | Out-Null
# Trigger name is the key in the saved definition; select the sole Request trigger.
$triggerNames=@($collectorResource.properties.definition.triggers.PSObject.Properties |
    Where-Object {$_.Value.type -eq 'Request'} | ForEach-Object {$_.Name})
if ($triggerNames.Count -ne 1) { throw 'Expected one collector Request trigger' }
$callback=AzJson @('rest','--method','POST','--url',"$ARM/providers/Microsoft.Logic/workflows/$($names.Collector)/triggers/$($triggerNames[0])/listCallbackUrl?api-version=2016-06-01")
$collectorUri=[UriBuilder]$callback.value
$collectorUri.Query=(($collectorUri.Query.TrimStart('?').Split('&') | Where-Object {$_ -notmatch '^(sig|sp|sv)='}) -join '&')
$COLLECTOR_URL=$collectorUri.Uri.AbsoluteUri
```

Use `$COLLECTOR_URL` in ADF. This is a workflow invocation URL, not an ARM management URL. The update preserves definition and connections; if you separately configured other access controls, merge them rather than discarding them.

**Validate:** anonymous POST is rejected; authenticated ADF invocation succeeds in Stage 12. Complete collection means eight completed groups and 72 actual member pairs. Unknown/non-user membership must fail the collection instead of silently disappearing.

## 12. Orchestrate assessment in Data Factory

In ADF Studio > Author > Pipelines > New, name `pl_rolepatrol_assess`, concurrency **1**. Add String parameter `BusinessAsOf`, default `2026-10-06T12:00:00`. Add String variables `RunId`, `CollectionId`, `CollectionStatus`.

Connect these activities with Success arrows:

| Order | Activity/configuration |
|---|---|
| 1 | Set Variable `Set_run`: RunId = `@guid()` |
| 2 | Stored Procedure `Validate_sources`: `ls_rolepatrol_sql`, `core.usp_Ingest` |
| 3 | Web `Start_collector`: collector OAuth URL, POST, body `{}`, Content-Type application/json, System Assigned Managed Identity, resource `https://management.azure.com/`; disable automatic async polling (`turnOffAsync=true`) |
| 4 | Set Variable `Save_collection`: CollectionId = `@activity('Start_collector').output.collectionId` |
| 5 | Until `Wait_for_collection`: `@or(equals(variables('CollectionStatus'),'Complete'),equals(variables('CollectionStatus'),'Failed'))`, timeout `00:10:00` |
| 6 | If Condition `Collection_complete`: `@equals(variables('CollectionStatus'),'Complete')`; false branch Fail |
| 7 | Stored Procedure `Build_assessment`: `core.usp_Assess`; RunId variable, BusinessAsOf pipeline parameter, InjectFailure false |
| 8 | Lookup `Check_run`: `core.usp_RunStatus`, RunId variable, First row only |
| 9 | If Condition `@equals(activity('Check_run').output.firstRow.Status,'Complete')`; false branch Fail |

Inside Until add sequentially:

1. Lookup `Poll_collection`: SQL linked service, stored procedure `core.usp_CollectionStatus`, CollectionId variable, First row only.
2. Set Variable CollectionStatus = `@activity('Poll_collection').output.firstRow.Status`.
3. Wait 5 seconds.

For a Lookup designer showing Query instead of procedure mode, use dynamic content `@concat('EXEC core.usp_CollectionStatus @CollectionId=''',variables('CollectionId'),''';')`. CollectionId must be the collector-generated GUID, never arbitrary user input. Use the equivalent query for Check_run. Never concatenate rationale/ticket text into SQL.

Click **Validate all**, fix every error, **Publish all**, then **Add trigger > Trigger now**. Keep the frozen BusinessAsOf. Open Monitor and inspect activity output. If Web output wraps JSON, adjust Save_collection to its actual collectionId path; missing ID must fail. Do not create a schedule yet.

After Succeeded, validate from Windows:

```powershell
Invoke-RpSqlFile (Join-Path $SUPPORT '03_validation.sql')
Invoke-RpSql 'SELECT * FROM core.Published; SELECT TOP(5) RunId,Status,BusinessAsOf,CompletedAt FROM core.Assessment ORDER BY StartedAt DESC;'
```

The validation script also injects a failed run and checks that publication stays unchanged; that Failed row is expected. Baseline UK Finance: **8 affected employees, 12 mature events, 10 raw matched tickets, 8 canonical tickets, 15/20 reporting peers**. Jane has conditional reporting loss; exceptions/independent assignment retain access; incomplete provenance is NeedsValidation.

**Connect the dots:** ADF coordinates; Logic Apps observes Entra; SQL calculates and publishes one complete evidence version. Frozen business time makes the scenario repeatable. Actual run/collection timestamps independently show freshness.

## 13. Wire the application to the real API

Proceed only after the Stage 7 browser authentication/CORS gate passes. Repeat for API A and B, each using its own SQL connection:

1. Keep the protected POST Request trigger. Replace the proof Response with SQL Execute stored procedure (V2), name `Gateway`, procedure `api.usp_Gateway`.
2. Payload = expression `string(triggerBody())`. Do not supply an actor field.
3. Add Response `Return_result` after success. Status = `int(first(body('Gateway')?['ResultSets']?['Table1'])?['HttpStatus'])`.
4. Body = `json(first(body('Gateway')?['ResultSets']?['Table1'])?['ResponseJson'])`.
5. Headers: Content-Type `application/json`, Cache-Control `no-store`, Access-Control-Allow-Origin exact `$APP_ORIGIN`, Vary `Origin`.
6. Add another Response after Gateway failed/timed out: 500, body `{"error":"Backend unavailable; reload or contact demo operator"}`, same headers. Do not reveal SQL/connector errors.
7. Save; verify actual SQL result-set casing and adjust expressions if necessary. Secure action inputs/outputs after troubleshooting.
8. Verify SAS remains disabled and all tenant/owner/client/scope claims remain intact. Keep the already-tested preflight arrangement: POST response headers alone do not implement OPTIONS.
9. Reload the Static Web Apps URL and sign in as Owner A. Overview now reads SQL data.

To change UI code, run `npm.cmd run build:app` and `npm.cmd run build:bridge` from the web folder, then `Build-Html.ps1 -ConfigFile <private config path>` and redeploy the `dist` folder to **Azure Static Web Apps** with `npx.cmd swa deploy ./dist --env production`. Never overwrite a finished API with the proof definition.

**Validate:** Owner A can open UK evidence/Jane; Owner B can see authorized aggregates but cannot retrieve Jane's details or decide her case. Changing IDs in JSON must not bypass SQL authorization. Use separate private browser profiles.

## 14. Half-time working-demo acceptance

| Test | Action | Required result |
|---|---|---|
| Ranking | Open Overview | UK Finance ranked by 8 distinct affected people; denominator/coverage visible |
| Evidence | Open UK role | Raw/canonical distinction and ticket references explain the result |
| Before-change warning | Open Jane | Reporting risk and versioned evidence; conditional language |
| Counterexamples | Inspect E023/E024/E025 | Exception / independent assignment / insufficient provenance |
| Decision | Owner A requests IAM review with rationale | Case updated; actor E029 and evidence version recorded |
| Unauthorized | Owner B requests Jane's detail/decision | Denied; no decision written |
| Duplicate | Replay exact successful request, same idempotency key | Same result; one decision |
| Conflicting retry | Change rationale with same key | Conflict; prior decision unchanged |
| Stale | Open case, assess again, submit old version | Conflict requiring reload |
| Browser-only | Use no Power BI account | Entire core journey works |

Use local browser DevTools for request replay; never upload tokens or HAR files. An Azure CLI token is not a substitute for a SPA token because the API validates the client application too.

```powershell
Invoke-RpSql 'SELECT TOP(10) * FROM casework.Decision ORDER BY CreatedAt DESC; SELECT TOP(20) * FROM audit.Event ORDER BY AuditId DESC;'
```

Record UTC time, RunId and pass/fail, without secrets. SQL impersonation proves SQL scope, not Entra sign-in. HTTP 200 alone does not prove correct numbers or idempotency.

**Milestone:** complete this journey before optional services. If browser feasibility failed, label SQL/API simulation as a restricted-lab fallback, not a finished manager application.

## 15. Validate history, peer insight and isolated what-if

### 15.1 Versioned assessment history

1. Record the current Overview RunId and UK count. Run the ADF pipeline again with the same frozen business time.
2. Select **History** in the same application. Both completed RunIds must appear; Failed test runs must not appear as selectable completed evidence.
3. Select the older run. Its ranking, tickets, memberships and projections must remain unchanged. The review-task status is explicitly current, not a historical reconstruction of case state.
4. Return to Overview/Refresh before submitting decisions. A decision records the case's evidence RunId and case version.
5. Execute the failed-publication validation in `03_validation.sql` again. Confirm latest-complete still points to the last successful assessment.

The assessment freezes normalized evidence plus SourceJson. It does not clone the whole database. Authorization remains current: an owner who loses permission must also lose access to old evidence. Administrators can technically alter SQL; the demo's immutability is enforced through application grants/transactions, not cryptographic or regulatory immutability.

### 15.2 What-if steps

1. Sign in as Owner A; choose current UK Finance evidence > **What-if preview**.
2. Entitlement `ENT_REPORT`, change `Add`, effective time `2026-10-12T12:00` UTC; Calculate.
3. Expect 5 potential recipients among the current 20 UK Finance members, and 15 already observed holders. This is not 5 guaranteed successful provisions.
4. Change entitlement to `ENT_FIN_INPUT`, change `Remove`; Calculate. Expect 20 potential losses with the baseline origin/exception fixture.
5. Try `ENT_ADMIN`: the policy notice must require specialist review, never suggest automatic approval.
6. Inspect `scenario.Result` and `audit.Event`; scenario calculations are recorded independently. Confirm original `src.RoleEntitlement`, live Entra membership and published ranking have not changed.

```powershell
Invoke-RpSql 'SELECT TOP(5) ScenarioId,RunId,RoleId,CreatedAt,ResultJson FROM scenario.Result ORDER BY CreatedAt DESC;'
```

The supplied scenario engine projects a single entitlement delta for the **current cohort**, using exceptions at the proposed effective time. It also returns a target-delta list for upcoming transfers; this is not a full recomputation of all future assignments. Do not claim broader simulation coverage. Stage 20 describes the work required before making that stronger claim.

### 15.3 Business explanation in the same UI

Keep these labels next to evidence, using fixed templates rather than a generative model:

| Evidence | Safe explanation / suggested next step |
|---|---|
| Repeated verified reporting complaints | Owner should review whether Reporting belongs in the role baseline; IAM validates application/context and conflicts |
| Conditional transfer loss | Validate continuing business need, provenance and transfer-time exceptions before requesting governed maintenance |
| Unknown provenance | Obtain missing assignment/exception evidence; no confident loss claim |
| Peers commonly hold an entitlement | Investigate a possible baseline gap; peer frequency does not establish entitlement legitimacy |
| Privileged entitlement | Specialist security/IAM review required |

The supplied UI includes ranking, peer tables and transfer predictions. Add these explanatory sentences in `web/app.js` with the existing `text('p',...)` helper, immediately after the corresponding table. Do not use `innerHTML` for ticket text. Rebuild both bundles and HTML as in Stage 7, redeploy the Static Web Apps `dist` package, then verify keyboard navigation and readability at 125% zoom.

For productivity context, use **scenario estimates**, not measured loss: 8 affected people × assumed 4 blocked hours = 32 illustrative hours; 8 canonical tickets × assumed 20 support minutes = 160 illustrative minutes. Show both assumptions in the same sentence. Ticket resolution duration is elapsed time and is not employee time lost. Do not sum overlapping ticket durations as productivity loss.

**Validation:** a leader can explain why UK Finance ranks first, why Jane needs review and why a peer majority is insufficient to authorize access, without opening SQL or Power BI.

## 16. Record IAM handoff and external outcome evidence

An owner decision is a request for governed review, not remediation. The reference app records an external IAM reference through **My work > Record IAM reference** for a principal with CanIam for that role.

The baseline deliberately does not grant Owner A IAM authority. For a demo where Owner B also acts as an IAM reviewer of UK cases, explicitly grant that role **after** the Stage 14 wrong-owner tests:

```powershell
Invoke-RpSql @'
UPDATE core.Permission SET CanReadDetail=1,CanIam=1,CanDecide=0
WHERE EmployeeId='E030' AND RoleId='R_FIN_UK';
INSERT audit.Event(Actor,ActionName,ObjectId,Detail)
VALUES(USER_NAME(),'DemoPermissionChange','E030','IAM detail/handoff access for R_FIN_UK; no owner decision permission');
'@
```

1. Owner A requests IAM review of Jane.
2. Owner B signs in, opens My work, finds Jane and records a fictional reference such as `MOCK-IAM-2026-001`.
3. Confirm the reference and audit entry appear. Owner B still cannot submit an owner decision for Jane.
4. Explain the role change in the demo: earlier Owner B was aggregate-only for UK; now explicitly acting as scoped IAM. Do not call its newly authorized detail access a security failure.
5. Restore baseline scope after this demonstration:

```powershell
Invoke-RpSql @'
UPDATE core.Permission SET CanReadDetail=0,CanIam=0,CanDecide=0
WHERE EmployeeId='E030' AND RoleId='R_FIN_UK';
INSERT audit.Event(Actor,ActionName,ObjectId,Detail)
VALUES(USER_NAME(),'DemoPermissionChange','E030','Restored aggregate-only R_FIN_UK access');
'@
```

**Outcome verification is optional and not automatically implemented by recording a reference.** The HLD distinguishes source-baseline correction from functional access restoration. If adding the optional outcome feature, implement this exact contract before showing a Verified status:

1. Setup operator, acting as the **external mock SailPoint source**, introduces a new effective-dated role version. Close the old version at the same instant; copy unchanged entitlements; add the proposed entitlement; update/split affected source assignments and planned move version references at the boundary. Do not edit the historical assessment or let the review API perform these source edits.
2. Run ingestion and a new assessment; stop on overlapping versions, invalid assignments or stale source coverage.
3. In a new `api.usp_VerifyOutcome` procedure, resolve actor from `core.Principal`, enforce scoped CanIam, lock the case/version, and require an assessment newer than the handoff. Read the expected new version/entitlement from that assessment's frozen source, not a client assertion.
4. Atomically insert `casework.Outcome` with type `RoleDefinitionUpdated`, evidence RunId/version, verifier and audit event. Add unique request-key protection if the action can be retried.
5. Expose it through `api.usp_Gateway` as a new explicitly allowed operation; add its button to the existing worklist. No new end-user tool is needed.
6. A separate `FunctionalAccessVerified` outcome requires explicit application-owner/user verification evidence; Entra group presence alone is insufficient. Without this evidence, leave functional verification pending.

**Validate:** unchanged source cannot produce RoleDefinitionUpdated; altered source without a fresh assessment cannot pass; wrong IAM scope is denied; duplicate requests create one outcome; old evidence is unchanged. Until this extension is implemented and these tests pass, show **Handoff requested / external reference recorded**, not Verified. This stage is not needed for the half-time core demo.

## 17. ServiceNow mock boundary

The supplied implementation uses synthetic `src.Ticket` rows and the in-app **Mock ServiceNow Review Task** panel. This satisfies the permitted synthetic-data option without deploying a second task website or contacting ServiceNow.

1. Open a role's evidence and check ticket identifiers/text against the synthetic SQL source as setup operator.
2. Open a review task. Display assignment/role owner, affected employee, evidence version, due date and rationale before submission.
3. Submit a decision through the same scoped API. Verify it changes only `casework`/`audit`, never source tickets or Entra.
4. Keep `Mock ServiceNow` visible in the panel title and demo narration. Do not imply a real ServiceNow integration or automatic maintenance request was sent.

There is no requirement to build a second mock REST facade when using the approved synthetic-data option. If a judging requirement specifically asks for a mock GET ticket endpoint, add a read operation to the existing gateway that enforces the same role-detail scope and selects the requested ticket from frozen evidence. Do not expose a public endpoint over all source tickets.

**Validation:** all end users remain in the browser application; no corporate HR, SailPoint or ServiceNow network call appears in any workflow.

## 18. Optional Azure AI Search: candidates only

Skip this stage until the core is stable. Azure AI Search can retrieve catalog candidates without an LLM; the allowlist does not include Azure OpenAI. No generative model is required for ranking, projection, authorization or decisions.

1. Azure Portal > Create Azure AI Search in the same group/region, **Free** tier if available. If not available, stop this optional branch unless you deliberately accept a permitted paid tier.
2. Open the service > Indexes > Add index, name `rp-entitlements`. Fields: `id` Edm.String key/filterable; `displayName` Edm.String searchable; `contextCode` Edm.String filterable; `risk` Edm.String filterable. Save.
3. Obtain the eight catalog rows from `SELECT * FROM core.vw_Catalog` as setup operator. Upload using the REST documents endpoint below. Keep the administration key only in a local SecureString prompt, never in the browser bundle or a committed file.

```powershell
$SEARCH_NAME = Read-Host 'Exact Search service name'
if ($SEARCH_NAME -notmatch '^[a-z0-9-]+$') { throw 'Invalid service name' }
$searchKey = Read-Host 'Search administration key (optional setup only)' -AsSecureString
$keyPtr = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($searchKey)
try {
    $keyText = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($keyPtr)
    $catalog = @(Invoke-RpSql 'SELECT EntitlementId,DisplayName,ContextCode,Risk FROM core.vw_Catalog;')
    $docs = @($catalog | ForEach-Object {
        @{'@search.action'='mergeOrUpload';id=$_.EntitlementId;displayName=$_.DisplayName;contextCode=$_.ContextCode;risk=$_.Risk}
    })
    $base = "https://$SEARCH_NAME.search.windows.net/indexes/rp-entitlements/docs"
    $headers = @{'api-key'=$keyText}
    $upload = Invoke-RestMethod -Method Post -Uri "$base/index?api-version=2024-07-01" -Headers $headers -ContentType 'application/json' -Body (@{value=$docs}|ConvertTo-Json -Depth 8)
    if (@($upload.value | Where-Object {-not $_.status}).Count) { throw 'One or more documents failed indexing' }
    $candidates = Invoke-RestMethod -Method Post -Uri "$base/search?api-version=2024-07-01" -Headers $headers -ContentType 'application/json' -Body '{"search":"reporting","top":3,"select":"id,displayName,contextCode,risk"}'
    $candidates.value | Select-Object id,displayName,contextCode,risk,@{n='RetrievalScore';e={$_.'@search.score'}}
} finally {
    [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($keyPtr)
    $keyText=$null; $headers=$null; $searchKey=$null
}
```

This is a setup-time search proof, not an automated ingestion pipeline. [Search document indexing API](https://learn.microsoft.com/en-us/rest/api/searchservice/documents/index?view=rest-searchservice-2024-07-01).

4. Run `EXEC core.usp_SearchQueue` as setup operator. Pick an actual unresolved ticket and a catalog candidate only after inspecting context. Store the retrieved candidate with the parameterized SQL procedure `core.usp_StoreCandidate` (TicketId, EntitlementId, optional Relevance). Do not pre-mark it Confirmed or convert its search score into a confidence percentage.
5. Grant the designated IAM reviewer CanIam/CanReadDetail for the ticket-time role through an audited setup change, as demonstrated in Stage 16. In the browser use **IAM ticket matches**, Confirm or Reject.
6. Run a new assessment. Confirmed candidates may contribute only when all transfer-window/type/context rules also pass. The baseline ambiguous ticket may not be eligible for the historical rank; confirmation must not bypass maturity eligibility.
7. Reopen the older assessment: its result must not change.

To automate retrieval later, use a Logic App with its own restricted SQL queue/candidate procedures and supported Search authentication. Do not give the owner API source-write rights. Free-tier identity/features must be verified before choosing managed identity rather than a securely stored connector credential.

**Validation:** unconfirmed candidates never alter ranking; catalog IDs outside the allowlist are rejected; ticket text is treated as data; the complete core works with Search disabled. Production adoption needs approved data minimization and retrieval-quality testing.

## 19. Optional Power BI for licensed analysts

All managers/leaders already have the application. Do not make this optional report a dependency for review.

1. Install Power BI Desktop from Microsoft's official distribution if permitted on the laptop. Azure trial does not supply Power BI service sharing licenses.
2. Create a dedicated Entra SQL user for the analyst (using its verified object ID and SID/TYPE method in Stage 10). Grant SELECT only on `core.vw_ImpactReporting`. Do not reuse the SQL administrator credential or grant source/evidence tables.
3. Power BI Desktop > Get data > Azure > Azure SQL Database. Enter the server/database, choose **Import**, and authenticate with the analyst's Organizational account. Select `core.vw_ImpactReporting`, Transform Data.
4. Keep only reporting columns; ensure RunId/RoleId are Text, BusinessAsOf/CompletedAt are DateTime, counts Whole number. Close & Apply.
5. Add a **single-select RunId slicer**. Add a table with RoleId, AffectedEmployees, EligibleEvents, CanonicalTickets and Coverage; set numeric columns to **Don't summarize** or use MAX measures. Sort AffectedEmployees descending. Do not sum the same role across different snapshots.
6. Add cards for BusinessAsOf and CompletedAt, with a visible `Select one assessment` label. Add a text box explaining distinct employees, mature transfer window and synthetic source data.
7. Add a Web URL button to the **Static Web Apps application URL**, never a bearer token or business API URL. Label `Open Role Patrol to review`.
8. Save the PBIX in the private build folder. Refresh after a new assessment; Import is not instant/live.
9. Publish only if the analyst has an approved workspace and required licensing. Share only with the authorized aggregate-analysis audience. This view contains all roles and has no per-user SQL row-level security: do not share it to scoped role owners as if it mirrored application authorization.

If scoped Power BI sharing is later required, implement and test dynamic RLS with an approved viewer-to-role table; test using actual Viewer users, not workspace administrators. That is additional work, not provided by the import connection. [Power BI sharing requirements](https://learn.microsoft.com/en-us/power-bi/collaborate-share/service-share-dashboards), [Power BI RLS](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security).

**Validation:** for the same RunId, SQL, app and report show the same UK values. An unlicensed manager completes the browser journey without opening Power BI.

## 20. Implementation scope, limitations and closure work

This reference build prioritizes the winning core. It does not claim every production control or optional HLD feature is finished. Use this table to decide what you can honestly demonstrate.

| HLD capability | Reference build | Work/acceptance before a broader claim |
|---|---|---|
| Distinct-employee role ranking | SQL calculation and scoped browser overview | Run baseline and negative fixtures in Azure; check tie and zero-denominator behavior |
| Frozen assessments | Transactional publication, SourceJson and evidence tables | Compare old/new runs; confirm failed publication leaves pointer unchanged |
| Transfer preview | Direct-group fixture, provenance/exception-aware outcomes | Not effective access across nested/dynamic groups, application policies or future provisioning failures |
| Role-edit what-if | Single-entitlement current-cohort delta, stored separately | Validate entitlement is actually absent/present in selected version; project assignment membership at effective time; recompute upcoming transfers, not just a target-delta label |
| Business-role/group gap | Compare baseline, observed peers and post-transfer evidence | Add a dedicated gap summary if presenting as a standalone finding family; agreement among peers alone is insufficient |
| Productivity | Ticket burden plus explicitly illustrative assumptions | Actual blocked hours require separately collected evidence; no measured savings claim |
| Browser authentication | MSAL bridge and protected owner-specific workflows | Mandatory hosting/CORS/conditional-access proof in this account; no universal success guarantee |
| Enterprise reviewer scale | Two owner-bound endpoints map to distinct SQL users | A scalable trusted identity-to-SQL context design needs a separate security review; do not provision a workflow per enterprise user |
| Review lifecycle | Owner decision, rationale, optimistic version, idempotency, audit, external reference | No automatic access change; optional verified outcomes need Stage 16 extension |
| Owner changes | Current permission table and case owner | Govern reassignment, increment case version and audit; no silent owner changes |
| Search | Candidate storage, scoped confirmation, optional retrieval proof | Automated retrieval orchestration and quality evaluation are optional extensions |
| Operations | Manual ADF runs, run history, SQL audit, current worklist | No automatic escalation delivery or holiday calendar; add only after core stability |
| Security hardening | Least-privilege procedure calls, OAuth, encoded text, secure logs | CSP/browser testing, retention policy, concurrency/load tests and independent review remain necessary before organizational deployment |

For the what-if extension, parse the selected assessment's versions/baseline/assignments into temporary tables. Reject Add if entitlement already exists, and Remove if absent. Build the cohort using assignment intervals containing effectiveAt, then evaluate target baseline plus proposed delta, origin removals and active exceptions per employee. Preserve Unknown when source coverage is incomplete. Compare unchanged vs proposed outcomes; store only in scenario tables. Validate Jane becomes retained by the proposed target when Reporting is added, and current-member/entrant/leaver totals remain separate. Do not call the current target-delta string a completed implementation of that algorithm.

For a baseline-gap display, show each catalog entitlement with `in role version`, `observed holders / cohort`, `verified affected employees`, `source freshness` and `suggested review step`. Fetch frozen baseline and evidence for the same RunId, enforce existing role scope, and render in the existing role panel. Use categories **Review baseline omission**, **Review unexpected access**, **Expected alignment**, **Insufficient evidence**, with explicit evidence conditions. Avoid adding an arbitrary risk score.

**Acceptance rule:** if an extension is unfinished, hide its incomplete button or label its bounded scope. The core can still be successful; an unimplemented feature is not a completed deliverable merely because its table/procedure name exists.

## 21. Final validation, reset and rehearsal

### 21.1 Negative-test checklist

Perform source overlays only in a dedicated test database or with a documented restore copy. Never alter already-published evidence to create a test outcome.

| Overlay/test | Expected behavior |
|---|---|
| New duplicate ticket for same complaint | Raw count changes; affected-employee count does not inflate |
| Two eligible moves for one person | Person counts once per role; event denominator remains separate |
| Transfer less than 30 days old | Excluded from mature role ranking, recent activity still explainable |
| Country/context mismatch | Does not establish the proposed local role entitlement need |
| No eligible movers | No misleading percentage or healthy rank |
| Partial ticket/source coverage | Partial/insufficient coverage, not confident rank |
| Graph 403 or one failed page | Failed/incomplete collection, never valid empty membership |
| Missing provenance/expired exception | Unknown or conditional outcome as rules require |
| Failed publication | Last complete snapshot remains available |
| Wrong tenant/client/reviewer | Trigger rejects request |
| Forged actor in body | Ignored; actor derives from runtime identity mapping |
| Two decisions using one case version | At most one new state transition; loser reloads |
| Retry same request/key | One decision and audit event |
| Script-like text in ticket/rationale | Displayed as text, never executed |

Some overlays deliberately invalidate the fixed 40-ticket fixture manifest. When testing a changed manifest, adjust the test-specific expected count in an isolated database; do not weaken production-like ingestion checks merely to accept malformed data. The supplied baseline validation is not a substitute for all these tests.

### 21.2 Reset without pretending history vanished

For a second demo, refresh source collection/assessment and use a new open case or an explicitly audited case reopen. Do not delete decisions/audit rows to make a completed case look untouched. The seed script intentionally refuses reinitialization.

For a genuinely clean rehearsal environment:

1. Export the existing synthetic assessment/decision/audit evidence if you need it; keep files in the private build folder.
2. Create a **new dedicated SQL database** using the checked free-offer configuration and a new name. Check offer capacity first.
3. Change `$DB`, rerun schema, identity-map, engine, operations and status scripts; recreate external users/grants/principal mappings.
4. Create/repoint each workflow's SQL connection and the ADF linked service to the new database. Save/publish.
5. Recollect real demo memberships, run ADF and rerun Stage 14. Do not reseed observations.
6. Keep the old database untouched until its retained evidence is no longer required, then delete it explicitly through Portal after checking its exact name.

This reset costs more setup time but preserves audit honesty. A future automated reset should reset only synthetic source fixtures and use a new run/case generation, with explicit reset audit entries.

### 21.3 Six-minute presentation

1. **0:00–0:40:** business problem: transfers generate avoidable access requests and lost working time.
2. **0:40–1:30:** Overview: UK Finance has 8 affected employees; explain why raw ticket volume is not the rank.
3. **1:30–2:30:** role evidence and Jane's imminent transfer; explain exact supporting evidence and one uncertainty counterexample.
4. **2:30–3:30:** owner submits a rationale-backed review request; show saved status and evidence version in the app.
5. **3:30–4:30:** isolated role-edit scenario; no access changes. State its current-cohort scope.
6. **4:30–5:20:** History: old snapshot remains reproducible. If implemented, show IAM handoff, not an invented verified outcome.
7. **5:20–6:00:** adoption: browser for managers/leaders/IAM, optional analyst BI; bounded AI use, no automatic remediation; distinguish observed burden from illustrative productivity estimates.

Before presenting, run the baseline checks, sign in both private profiles, wake serverless SQL, confirm recent collection and disable unneeded ADF triggers. Keep redacted screenshots of the **actual successful run** as a disclosed fallback; never present them as live execution.

## 22. Troubleshooting, spending control and cleanup

| Symptom | First check / next action |
|---|---|
| Azure operation forbidden | Exact subscription/tenant and resource provider registration; then assigned Azure role/policy |
| Cannot create user/app/Graph grant | Tenant directory role/consent; subscription ownership is insufficient |
| AADSTS redirect mismatch | Exact registered SPA bridge URL including query; no trailing-character changes |
| Popup succeeds but API 401 | API audience/version, issuer, tenant, reviewer oid, SPA azp and delegated scope |
| Browser CORS failure | Inspect OPTIONS independently; follow Stage 7 stop condition |
| SQL login failure | Managed identity principal ID/SID, target database and workflow-specific connection |
| SQL timeout | Serverless wake-up, firewall ranges and integration-runtime region; retry after confirmed readiness |
| Procedure not listed | Schema-qualified custom value/metadata discovery; do not grant db_owner |
| Collector 403 | Collector Graph app-role assignment and token propagation; no write permission needed |
| ADF waits forever | Collector run failure, output collectionId path, status procedure and timeout |
| Numbers differ | Frozen clock, source versions, duplicate roots, eligibility windows and observed membership |
| UI still shows old data | Refresh app; select latest completed run. Power BI Import refresh is separate |

Check Cost Management daily and after optional service creation. Keep pipeline triggers manual. Disable workflows/pipelines when not demonstrating; disabling workflows does not delete SQL/Search or eliminate their costs. SQL serverless pausing depends on configuration/activity; inspect status instead of assuming it has paused.

Cleanup is deliberately explicit:

1. Save required synthetic evidence, resource names and redacted deployment definitions. Never save tokens, passwords or API keys in the shared project folder.
2. Assert the exact subscription with `Assert-RpContext`. In Portal open the saved dedicated resource group and inspect every resource.
3. If all resources in it belong solely to this demo, select **Delete resource group**, enter the exact group name and confirm. This deletes SQL data and workflows; do it only after the demonstration/evidence-retention need has ended.
4. Resource-group deletion does **not** delete the created Entra users/groups or app registrations. Use the saved identity/group maps and exact SPA/API application IDs; verify each object's demo prefix/object ID in Entra before deleting those demo objects. Do not bulk-delete by a broad name pattern.
5. Remove the optional Power BI report/dataset from its workspace and Search keys/configuration files if no longer needed. Remove the laptop SQL firewall rule if the SQL server remains for any reason.
6. Check resource group deletion, remaining Entra demo objects and billing later for final usage posting. Trial expiration is not a substitute for cleanup.

**Final validation:** record which Stage 14 tests passed, which optional capabilities were built, which Stage 20 limitations remain and which resources remain billable. This document supplies implementation instructions and reference code; Azure execution results must come from your account.

## Appendix: files and execution order

Author-side checks: all three supplied PowerShell scripts and all 30 fenced PowerShell blocks were parsed successfully; package/config JSON parsed and the 22-stage structure was checked. The local execution policy blocked running the workflow generator, so generated ARM bodies were not execution-validated here. Node/SQL runtime and Azure account tests were not performed. Run the stage validations before calling the deployment successful.

| File | Purpose / execution |
|---|---|
| `New-DemoDirectory.ps1` | Setup-only fictional users/groups and mapping files; Stage 5 |
| `New-WorkflowBody.ps1` | Protected API proof ARM resource bodies; Stage 7. The legacy shell generator is not used as OAuth hosting. |
| `web/package.json`, `app.js`, `bridge.js`, `Build-Html.ps1` | Locally bundled browser application, MSAL return bridge and Static Web Apps deployment package |
| `01_schema_seed.sql` | New-database schema and deterministic synthetic sources; once |
| Generated private `identity-map.sql` | Maps synthetic IDs to actual demo Entra objects |
| `02_engine.sql` | Assessment, scope, decisions, scenarios and baseline validation |
| `04_operations.sql` | Ingestion checks, candidate confirmation, gateway and reporting views |
| `05_status.sql` | ADF status polling procedures |
| `03_validation.sql` | Baseline assertions and failed-publication test after a successful pipeline |

Order: **01 → private identity map → 02 → 04 → 05 → grants/connections → collector/ADF → 03 → browser acceptance**. Reapplying procedure scripts is supported; reseeding a used database is not.


| Milestone | High-level tasks | Outcome |
|---|---|---|
| **1. Prepare and control the environment** | Review free-trial limits, install local tools, validate the exact Azure subscription, tenant and region. | Safe build environment with controlled costs. |
| **2. Establish Azure foundation** | Register providers, create the dedicated resource group and save generated resource names. | Isolated Azure workspace for Role Patrol. |
| **3. Create demo identities** | Create 30 fictional Entra users, eight security groups and 72 group memberships. | Realistic Entra test directory without corporate data. |
| **4. Set up browser authentication** | Create API and browser app registrations, delegated API scope, consent and Static Web Apps redirect URI. | Secure Entra PKCE authentication design. |
| **5. Deliver the fast browser card** | Deploy the Logic Apps HTML review card and its simulated-decision action against the reviewed synthetic case. | A browser-based, labelled fallback demo is available before SPA work. |
| **6. Host the enhanced browser portal** | Create Azure Static Web Apps, build the HTML/JavaScript bundle and deploy `index.html` and `bridge.html`. | Stable, non-secret portal URL for managers and leaders. |
| **7. Prove protected API access** | Deploy owner-bound Logic Apps proof APIs; test OAuth, cross-owner denial and CORS preflight. | Proof that the enhanced portal can securely call Logic Apps. |
| **8. Build the data foundation** | Create Entra-only Azure SQL, configure restricted firewall paths and load synthetic HR, role, mover and ticket data. | Secure, repeatable synthetic data store. |
| **9. Build the analysis engine** | Install SQL schema/procedures for ranking, evidence, transfer-risk predictions, snapshots, audit and decisions. | Deterministic role-health calculations. |
| **10. Connect service identities** | Create Data Factory, Logic Apps collector and managed-identity SQL permissions. | Services can call only the procedures they need. |
| **11. Collect Entra observations** | Grant the collector read-only Graph access and build the membership collection Logic App. | Live membership snapshot of fictional Entra groups. |
| **12. Orchestrate assessment** | Build the Data Factory pipeline to validate sources, collect memberships, run assessment and publish a snapshot. | Ranked role assessment with versioned evidence. |
| **13. Deliver the half-time demo** | Use the Logic Apps card to show ranking → evidence → labelled simulated review → audit. | Working browser demo even if the enhanced portal is incomplete. |
| **14. Deliver authenticated portal review** | Wire Static Web Apps to real protected APIs; test ranking → evidence → owner decision → audit. | Full end-to-end manager browser demo. |
| **15. Add business insight** | Add assessment history, peer evidence, isolated what-if preview and clear business explanations. | Better owner and leadership decision support. |
| **16. Add operational handoff** | Record IAM handoff references and optionally implement verified external outcomes. | Review decisions can be tracked without automatic remediation. |
| **17. Add optional enhancements** | Add Azure AI Search candidate retrieval and Power BI analyst reporting only after core stability. | Extra value without risking the core demo. |
| **18. Validate, rehearse and clean up** | Run negative/security tests, rehearse the six-minute demo, monitor spend and remove resources when finished. | Credible, repeatable presentation with controlled cost. |

The first working-demo checkpoint is milestone **13**. The authenticated Static Web Apps portal is milestone **14**. Milestones **15–18** are added only after both UI paths are stable.
