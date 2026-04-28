# GetDefenderExpertsTasks – Defender Experts Managed Response Task Notifications (Logic App)

> **Disclaimer**
> This repository is a community-driven sample and is **not** an official, supported Microsoft product. It is provided "as is" with no warranties, guarantees, or support obligations of any kind, and its use is entirely at your own risk.

This solution deploys an Azure Logic App (Consumption) that monitors Microsoft Defender Experts (DEX) **Managed Response** tasks (Microsoft Graph Security API) and sends notifications when new tasks are observed.

## What it does

- Polls Microsoft Graph for Defender incident tasks on a schedule
- De-duplicates tasks using Azure Table Storage (so you don’t alert twice for the same task)
- Enriches each incident with its details and related alerts
- Sends a rich HTML email (Office 365 Outlook connector)
- Creates a ServiceNow record (ServiceNow connector)
- Writes processed task IDs to Azure Table Storage for auditing/idempotency

## What it does NOT do

The current deployed workflow in [deploy/logicapp.json](deploy/logicapp.json) **does not**:

- Post Teams adaptive cards
- Reset passwords / revoke user sessions
- Mark incident tasks as completed in Microsoft Graph (PATCH)

If you want any of the above behaviors, you’ll need to extend the workflow definition (see “Optional enhancements”).

## Components

- Azure Logic App (Consumption)
  - Trigger: recurrence (default every 5 minutes)
  - System-assigned managed identity is used for Microsoft Graph HTTP actions
- Microsoft Graph Security API
  - `GET /beta/security/incidentTasks`
  - `GET /beta/security/incidents/{id}?$expand=alerts`
- Azure Storage Table
  - Table: `DefenderIncidentTasksProcessed`
  - `PartitionKey = incidentId`, `RowKey = taskId`
- Managed API connections
  - `azuretables` (configured for Managed Identity auth)
  - `office365`
  - `service-now`

## Workflow overview (current template)

1. **Recurrence trigger** runs every 5 minutes.
2. **Time window overlap** computes `TimeWindowStart = utcNow() - 15 minutes`.
3. **Get incident tasks** calls:
   - `GET https://graph.microsoft.com/beta/security/incidentTasks`
   - Uses an OData filter: base filter + `lastModifiedDateTime ge {TimeWindowStart}`
4. **De-dupe** each task against Azure Table Storage.
   - If there is no row for `(incidentId, taskId)`, the task is treated as “new”.
5. **Enrich incidents** by querying each distinct incident ID with `$expand=alerts`.
6. **Notify**
   - Builds HTML blocks for tasks + alerts
   - Sends email via Office365 connector
   - Creates a ServiceNow record
7. **Persist** processed tasks to Azure Table Storage (`DefenderIncidentTasksProcessed`).
8. **Catch scope**
   - Sends a failure email and terminates the run as Failed.

## Deployment

There are two ARM templates and they must be deployed in this order:

1. **Storage account + Table**
2. **Logic App + API connections**

### One-click deployment

Deploy storage first:

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FSebmolendijk%2FGetDefenderExpertsTasks%2Fmain%2Fdeploy%2Ftablestorage.json" target="_blank">
  <img src="https://aka.ms/deploytoazurebutton" alt="Deploy to Azure" />
</a>

Then deploy the Logic App:

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FSebmolendijk%2FGetDefenderExpertsTasks%2Fmain%2Fdeploy%2Flogicapp.json" target="_blank">
  <img src="https://aka.ms/deploytoazurebutton" alt="Deploy to Azure" />
</a>

> If you fork the repo, update the GitHub owner/repo in the URLs.

### Deploy via Azure CLI (alternative)

1. Deploy storage:

```sh
az deployment group create \
  --resource-group <your-rg> \
  --template-file deploy/tablestorage.json \
  --parameters storageAccountName=<globally-unique-storage-name>
```

2. Deploy Logic App + connections:

```sh
az deployment group create \
  --resource-group <your-rg> \
  --template-file deploy/logicapp.json \
  --parameters \
    workflowName=GetDefenderExpertsTasks \
    location=<azure-region> \
    storageAccountName=<same-storage-name>
```

> The workflow is deployed in a **Disabled** state by default. Enable it in the Azure portal when you’re ready.

## Post-deployment configuration

### 1) Microsoft Graph permissions (Cloud Shell / PowerShell)

The Logic App uses a **system-assigned managed identity** to call Microsoft Graph.
Grant Graph application permissions by assigning Microsoft Graph **app roles** to the managed identity.

This repo’s guidance assumes you will do Graph permissions from **Azure Portal Cloud Shell (PowerShell)**.

#### 1.1 Open Cloud Shell (PowerShell)

1. Go to https://portal.azure.com
2. Click **Cloud Shell** (top-right)
3. Choose **PowerShell**

#### 1.2 Get the Logic App managed identity object ID

```powershell
# --- REQUIRED VALUES ---
$SubscriptionId = "<subscription-id>"
$ResourceGroup  = "<resource-group>"
$LogicAppName   = "<logic-app-name>"

az account set --subscription $SubscriptionId

# This is the Entra ID service principal object id for the system-assigned managed identity
$logicAppPrincipalId = az resource show --resource-group $ResourceGroup --name $LogicAppName --resource-type "Microsoft.Logic/workflows" --query identity.principalId -o tsv
$logicAppPrincipalId
```

#### 1.3 Connect to Microsoft Graph PowerShell

```powershell
Install-Module Microsoft.Graph -Scope CurrentUser -Force -AllowClobber

# Needs an account that can assign app roles (e.g., Global Admin / Cloud App Admin / Application Admin)
Connect-MgGraph -Scopes "Application.Read.All","AppRoleAssignment.ReadWrite.All"
```

#### 1.4 Get the Microsoft Graph service principal

```powershell
$graphSp = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"
$graphSp.Id
```

#### 1.5 Assign the required app roles

**Required for this workflow (read incident tasks + incidents):**

- `SecurityIncident.Read.All`

**Common additional permission (if `$expand=alerts` returns 403):**

- `SecurityAlert.Read.All`

**Optional (only if you add PATCH/update behavior to the workflow):**

- `SecurityIncident.ReadWrite.All`

```powershell
$securityIncidentRead = $graphSp.AppRoles | Where-Object {
  $_.Value -eq "SecurityIncident.Read.All" -and $_.AllowedMemberTypes -contains "Application"
}

$securityAlertRead = $graphSp.AppRoles | Where-Object {
  $_.Value -eq "SecurityAlert.Read.All" -and $_.AllowedMemberTypes -contains "Application"
}

$securityIncidentReadWrite = $graphSp.AppRoles | Where-Object {
  $_.Value -eq "SecurityIncident.ReadWrite.All" -and $_.AllowedMemberTypes -contains "Application"
}

# Assign required role
New-MgServicePrincipalAppRoleAssignment `
  -ServicePrincipalId $logicAppPrincipalId `
  -PrincipalId $logicAppPrincipalId `
  -ResourceId $graphSp.Id `
  -AppRoleId $securityIncidentRead.Id

# Optional: only if you need it
# New-MgServicePrincipalAppRoleAssignment `
#   -ServicePrincipalId $logicAppPrincipalId `
#   -PrincipalId $logicAppPrincipalId `
#   -ResourceId $graphSp.Id `
#   -AppRoleId $securityAlertRead.Id

# Optional: only if you add PATCH/update logic
# New-MgServicePrincipalAppRoleAssignment `
#   -ServicePrincipalId $logicAppPrincipalId `
#   -PrincipalId $logicAppPrincipalId `
#   -ResourceId $graphSp.Id `
#   -AppRoleId $securityIncidentReadWrite.Id
```

#### 1.6 Verify assignments

```powershell
$assignments = Get-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $logicAppPrincipalId |
  Where-Object { $_.ResourceId -eq $graphSp.Id }

$assignments | Select-Object AppRoleId, ResourceId, CreatedDateTime

# Show role values for readability
$graphSp.AppRoles |
  Where-Object { $_.Id -in $assignments.AppRoleId } |
  Select-Object Value, Id
```

> **Note:** It can take a few minutes for new permissions to take effect.

### 2) Storage permissions (Azure Portal)

The Azure Tables connector uses **managed identity** and needs a data-plane role assignment.

Assign this role on the **Storage Account**:

- Role: **Storage Table Data Contributor**
- Assignee: the Logic App’s **system-assigned managed identity**

Steps:

1. Open the **Storage account** you deployed.
2. Go to **Access control (IAM)**.
3. Select **Add** → **Add role assignment**.
4. Role: **Storage Table Data Contributor**.
5. Assign access to: **Managed identity**.
6. Select members:
   - Managed identity type: **Logic App**
   - Select your Logic App name
7. Click **Review + assign**.

### 3) Authorize API connections (Azure Portal)

After deployment, go to the Resource Group → open each API connection:

> The `azuretables` connection should already be configured for **Managed identity** by the ARM template, but it will still fail until you complete the Storage role assignment above.

- `office365`
  - Click **Edit API connection** → **Authorize**
- `service-now`
  - Configure/authorize to your ServiceNow instance (depends on your ServiceNow auth model)

#### ServiceNow record creation (recommended configuration)

> ⚠️ **Important:** The current workflow uses a **basic example** action that creates an **Incident** record in the ServiceNow **`incident`** table.
>
> Recommendations:
>
> - For security operations, many customers prefer creating records in a **dedicated security incident process/table** (for example, a Security Incident table/module) with **restricted ACLs** and scoped permissions.
> - The sample action populates a minimal set of fields and primarily copies the notification content (incident summary, tasks, and links). It does **not** set severity/priority/assignment fields in a standardized way.
> - Because every ServiceNow instance is configured differently (custom tables, fields, assignment rules, priorities, and workflows), treat this repo as a starting point and implement record creation using your **ServiceNow best practices**:
>   - Choose the correct table/process for security incidents
>   - Map Microsoft Defender severity/status to your ServiceNow fields
>   - Apply assignment groups, categorization, SLAs, and dedup/correlation rules
>   - Ensure the connector identity has least-privilege access to only the required table/fields

##### Enriched incident example (payload you can map in ServiceNow)

Inside the workflow, the ServiceNow action runs within the `For each enriched incident` loop. The object available there is the **enriched incident**: the Microsoft Graph incident details (with alerts expanded) plus an extra `tasks` property added by the workflow.

Use this shape to understand what you can map into ServiceNow fields (severity, category, MITRE, etc.). Your tenant’s Graph alert schema can vary by product and API version, so treat this as an example and confirm the exact properties in your Logic App run history.

Example (simplified):

```json
{
  "id": "12345678",
  "displayName": "Suspicious inbox rule created",
  "severity": "high",
  "status": "active",
  "assignedTo": "",
  "createdDateTime": "2026-04-28T09:30:00Z",
  "lastUpdateDateTime": "2026-04-28T09:44:00Z",
  "incidentWebUrl": "https://security.microsoft.com/incidents/12345678",
  "systemTags": ["defenderExperts"],
  "summary": "<incident summary html/text>",
  "alerts": [
    {
      "id": "da637b0c-...",
      "title": "Inbox rule created to forward mail",
      "severity": "high",
      "status": "active",
      "category": "SuspiciousActivity",
      "productName": "Microsoft Defender for Office 365",
      "alertWebUrl": "https://security.microsoft.com/alerts/...",

      "mitreTechniques": ["T1114"],
      "mitreTactics": ["Collection"],

      "recommendedActions": "<string or array depending on alert type>",
      "description": "<alert description>"
    }
  ],
  "tasks": [
    {
      "id": "task-001",
      "displayName": "Containment action requested",
      "description": "Defender Experts requested ...",
      "status": "open",
      "actionType": "text",
      "source": "defenderExpertsGuidedResponse",
      "lastModifiedByDisplayName": "Defender Experts",
      "lastModifiedDateTime": "2026-04-28T09:42:00Z",
      "responseAction": {
        "identifierValue": "user@contoso.com"
      },
      "incident": {
        "id": "12345678"
      }
    }
  ]
}
```

Common mapping examples (in the ServiceNow action body):

- Defender incident severity: `items('For_each_enriched_incident')?['severity']`
- Defender incident URL: `items('For_each_enriched_incident')?['incidentWebUrl']`
- First alert category/product: `first(items('For_each_enriched_incident')?['alerts'])?['category']` / `first(items('For_each_enriched_incident')?['alerts'])?['productName']`

If you want to map MITRE tactics/techniques, first confirm which properties exist on the alert objects in your environment (for example `mitreTechniques`, `mitreTactics`, or similar fields), then aggregate them across `alerts`.

### 4) Enable the workflow

The workflow deploys **Disabled**.

1. Open the Logic App
2. Click **Enable**
3. Run a manual test run (optional)

## Configuration & customization

### Notification recipient

In the current template, the email recipient is set as a workflow variable in the `Initialize_variables` action:

- Variable name: `notificationEmail`

- Default: `security@mycompany.xyz`
- Change it in the Logic App designer after deployment, or edit [deploy/logicapp.json](deploy/logicapp.json) before deployment.

### Base incident task filter

The base `$filter` portion is stored as a workflow variable in `Initialize_variables`:

- Variable name: `incidentTasksBaseFilter`

- Default: `(status eq 'open' or status eq 'inProgress')`
- The workflow automatically appends the time window condition (`lastModifiedDateTime ge ...`).

## Optional enhancements

- **Mark tasks completed in Graph**
  - Add a `PATCH https://graph.microsoft.com/beta/security/incidentTasks/{taskId}` action after notifications succeed
  - Grant `SecurityIncident.ReadWrite.All`

## Operational notes

- **Idempotency:** each `(incidentId, taskId)` is recorded in `DefenderIncidentTasksProcessed`.
- **Overlapping window:** the 15-minute overlap reduces risk of missing tasks during transient failures.
- **Least privilege:** start with `SecurityIncident.Read.All` and only add other roles if needed.
