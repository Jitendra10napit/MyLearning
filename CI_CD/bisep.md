Below is one **Bicep** file that sets up a typical .NET Web API environment on Azure, which is what most architect interviews expect you to know. After it come a parameter file, modules, deployment commands and interview talking points.

## What we'll create (and why each piece matters)

| Resource | Why you need it |
|---|---|
| **App Service Plan + Web App** | Hosts the .NET API |
| **Staging slot** | Blue-green deployment with a swap |
| **Managed Identity** | The app reaches Azure resources **without passwords** |
| **Key Vault** | Stores secrets; the app reads them via Key Vault references |
| **Application Insights + Log Analytics** | Monitoring, logs, health gates |
| **Storage Account** | Blobs, files, queues; Azure Functions also need it |
| **Azure SQL** | The database |
| **Role assignments (RBAC)** | Give the app's identity access to Key Vault and Storage |

## 1. Bicep basics (quick table)

| Keyword | Meaning |
|---|---|
| `param` | Input value, like a method parameter |
| `var` | Local calculated value |
| `resource` | An Azure resource to create |
| `module` | Calls another Bicep file, for reuse |
| `output` | Value returned after deployment |
| `@secure()` | Hides the value from logs (passwords) |
| `@allowed([...])` | Restricts input values |
| `uniqueString()` | Generates a unique name suffix |
| `existing` | Refers to a resource that already exists |

## 2. `main.bicep`

```bicep
// ================= PARAMETERS =================
@allowed([ 'dev', 'qa', 'prod' ])
param environment string

param location string = resourceGroup().location
param appName string = 'myapi'

@allowed([ 'B1', 'S1', 'P1v3' ])
param skuName string = 'B1'

param sqlAdminLogin string
@secure()
param sqlAdminPassword string

// ================= VARIABLES =================
var suffix          = uniqueString(resourceGroup().id)   // makes names globally unique
var planName        = 'plan-${appName}-${environment}'
var webAppName      = 'app-${appName}-${environment}-${suffix}'
var kvName          = 'kv-${appName}-${environment}-${take(suffix, 6)}'
var storageName     = 'st${appName}${environment}${take(suffix, 8)}'   // lowercase, no dashes
var sqlServerName   = 'sql-${appName}-${environment}-${suffix}'
var tags = {
  environment: environment
  application: appName
  managedBy: 'bicep'
}

// ================= MONITORING =================
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: 'log-${appName}-${environment}'
  location: location
  tags: tags
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 30
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-${appName}-${environment}'
  location: location
  tags: tags
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
  }
}

// ================= APP SERVICE PLAN =================
resource appServicePlan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: planName
  location: location
  tags: tags
  sku: { name: skuName }
  kind: 'linux'
  properties: {
    reserved: true          // required for Linux
  }
}

// ================= WEB APP =================
resource webApp 'Microsoft.Web/sites@2023-12-01' = {
  name: webAppName
  location: location
  tags: tags
  identity: {
    type: 'SystemAssigned'  // Managed Identity – no passwords
  }
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|10.0'
      alwaysOn: skuName != 'B1' ? true : false
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
      healthCheckPath: '/health'
      appSettings: [
        {
          name: 'ASPNETCORE_ENVIRONMENT'
          value: environment == 'prod' ? 'Production' : 'Development'
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
        {
          // Key Vault reference – the app reads the secret at runtime
          name: 'ConnectionStrings__DefaultConnection'
          value: '@Microsoft.KeyVault(VaultName=${kvName};SecretName=SqlConnectionString)'
        }
      ]
    }
  }
}

// Staging slot for blue-green (needs Standard tier or higher)
resource stagingSlot 'Microsoft.Web/sites/slots@2023-12-01' = if (skuName != 'B1') {
  parent: webApp
  name: 'staging'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|10.0'
    }
  }
}

// ================= KEY VAULT =================
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: kvName
  location: location
  tags: tags
  properties: {
    tenantId: subscription().tenantId
    sku: { family: 'A', name: 'standard' }
    enableRbacAuthorization: true     // use RBAC instead of access policies
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
  }
}

// Store the SQL connection string as a secret
resource sqlSecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: keyVault
  name: 'SqlConnectionString'
  properties: {
    value: 'Server=tcp:${sqlServer.properties.fullyQualifiedDomainName},1433;Database=${sqlDb.name};User ID=${sqlAdminLogin};Password=${sqlAdminPassword};Encrypt=True;'
  }
}

// Give the Web App permission to READ secrets
var kvSecretsUserRoleId = '4633458b-17de-408a-b874-0445c86b69e6'   // Key Vault Secrets User

resource kvRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, webApp.id, kvSecretsUserRoleId)
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', kvSecretsUserRoleId)
    principalId: webApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// ================= STORAGE ACCOUNT =================
resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  tags: tags
  sku: { name: environment == 'prod' ? 'Standard_ZRS' : 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: storage
  name: 'default'
}

resource container 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = {
  parent: blobService
  name: 'uploads'
}

// Web App can read/write blobs using Managed Identity
var blobContributorRoleId = 'ba92f5b4-2d11-453d-a403-e96b0029c9fe'  // Storage Blob Data Contributor

resource storageRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storage.id, webApp.id, blobContributorRoleId)
  scope: storage
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', blobContributorRoleId)
    principalId: webApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// ================= AZURE SQL =================
resource sqlServer 'Microsoft.Sql/servers@2021-11-01' = {
  name: sqlServerName
  location: location
  tags: tags
  properties: {
    administratorLogin: sqlAdminLogin
    administratorLoginPassword: sqlAdminPassword
    minimalTlsVersion: '1.2'
  }
}

resource sqlDb 'Microsoft.Sql/servers/databases@2021-11-01' = {
  parent: sqlServer
  name: 'db-${appName}'
  location: location
  tags: tags
  sku: {
    name: environment == 'prod' ? 'S1' : 'Basic'
  }
}

// Allow Azure services (App Service) to reach SQL
resource sqlFirewall 'Microsoft.Sql/servers/firewallRules@2021-11-01' = {
  parent: sqlServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

// ================= OUTPUTS =================
output webAppName string = webApp.name
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output keyVaultName string = keyVault.name
```

## 3. Parameter file per environment: `main.dev.bicepparam`

```bicep
using './main.bicep'

param environment = 'dev'
param skuName = 'B1'
param sqlAdminLogin = 'sqladminuser'
param sqlAdminPassword = readEnvironmentVariable('SQL_ADMIN_PASSWORD')  // never hard-code
```

Create `main.prod.bicepparam` the same way with `environment = 'prod'` and `skuName = 'P1v3'`. **Same template, different parameters per environment.**

## 4. Modules (reuse)

When the file grows, split it into modules:

```
infra/
├── main.bicep
├── modules/
│   ├── appservice.bicep
│   ├── keyvault.bicep
│   └── sql.bicep
```

```bicep
// in main.bicep
module kv 'modules/keyvault.bicep' = {
  name: 'keyVaultDeploy'
  params: {
    name: kvName
    location: location
  }
}

// use its output
output kvUri string = kv.outputs.vaultUri
```

Modules can also be shared across teams through a **Bicep registry in ACR** (`br:myacr.azurecr.io/bicep/keyvault:v1`), which is the infrastructure equivalent of shared YAML templates.

## 5. Deploying it

**Azure CLI:**

```bash
az group create -n rg-myapi-dev -l centralindia

# Preview changes first (like terraform plan)
az deployment group what-if \
  -g rg-myapi-dev \
  -f main.bicep \
  -p main.dev.bicepparam

# Deploy
az deployment group create \
  -g rg-myapi-dev \
  -f main.bicep \
  -p main.dev.bicepparam
```

**From the pipeline:**

```yaml
- stage: Infra_Dev
  jobs:
  - job: DeployInfra
    steps:
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'azure-dev'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az deployment group what-if  -g rg-myapi-dev -f infra/main.bicep -p infra/main.dev.bicepparam
          az deployment group create   -g rg-myapi-dev -f infra/main.bicep -p infra/main.dev.bicepparam
      env:
        SQL_ADMIN_PASSWORD: $(SqlAdminPassword)   # from Key Vault-linked variable group
```

The order is **Infra stage → Build → Deploy app**, or infrastructure in its own pipeline because it changes less often.

## 6. Other resources worth knowing briefly

| Resource | Bicep type | Interview point |
|---|---|---|
| **Function App** | `Microsoft.Web/sites` with `kind: 'functionapp,linux'` | Needs a storage account. Use the **Flex Consumption** plan for new apps because the Linux Consumption (Y1) plan is being retired. Durable Functions use the same resource. |
| **Container Registry** | `Microsoft.ContainerRegistry/registries` | Give the app the **AcrPull** role through Managed Identity, and disable the admin user. |
| **Container Apps** | `Microsoft.App/managedEnvironments` + `Microsoft.App/containerApps` | Revisions and traffic split for canary. |
| **Service Bus** | `Microsoft.ServiceBus/namespaces` + `/queues` or `/topics` | Event-driven microservices. Relates to your case re-sharing work. |
| **Front Door / App Gateway** | `Microsoft.Cdn/profiles`, `Microsoft.Network/applicationGateways` | WAF, global routing, TLS termination. |
| **Private Endpoints + VNet** | `Microsoft.Network/privateEndpoints` | Production security: SQL, Key Vault and Storage reachable only privately. |

## 7. Production improvements you should mention

- **Entra ID (Azure AD) authentication for SQL** instead of a SQL password: the app connects with its Managed Identity, so no connection-string password exists at all. The SQL password above is kept only to make the example simple.
- **Private endpoints** and disabled public network access for SQL, Key Vault and Storage.
- **Diagnostic settings** that send all resource logs to Log Analytics.
- **Resource locks** (`CanNotDelete`) on production resources.
- **Azure Policy** to enforce tags, allowed regions, HTTPS-only and similar rules.
- **Autoscale rules** on the App Service Plan in production.

## 8. Bicep vs. ARM vs. Terraform (common question)

| | Bicep | ARM JSON | Terraform |
|---|---|---|---|
| Language | Simple DSL | Verbose JSON | HCL |
| Cloud | Azure only | Azure only | Multi-cloud |
| State file | **No** (Azure is the state) | No | **Yes** (must store and lock it) |
| New Azure features | Available on day 1 | Day 1 | Sometimes delayed |
| Preview | `what-if` | `what-if` | `plan` |

Bicep **compiles to ARM JSON**.

## Short interview line

"We provisioned all environments with **Bicep**: one `main.bicep` with **modules**, and **one parameter file per environment**. The Web App uses a **system-assigned Managed Identity**, gets **RBAC roles** on Key Vault and Storage, and reads secrets through **Key Vault references**, so there are no passwords in config. The pipeline runs **`what-if` first**, then deploys, so every change is reviewed and repeatable, and no one creates resources manually in the portal."
