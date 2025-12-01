# Part 4 — Infrastructure as Code

## Why This Matters

Infrastructure as Code (IaC) treats your cloud resources like software: version-controlled, reviewed, tested, and reproducible. No more "I made that change in the portal."

> "If it's not in code, it doesn't exist. If you can't recreate it from scratch, you don't own it." — DevOps proverb

Benefits of IaC:
- **Reproducibility**: Same config = same environment, every time
- **Auditability**: Git history shows who changed what and when
- **Collaboration**: Changes are reviewed via pull requests
- **Disaster recovery**: Recreate entire environments from code

---

## What Goes Wrong Without This

### The Snowflake Environment

```
Developer: "Production is slow, let's check the config"
Ops: "The App Service is Standard S3"
Developer: "Staging is Standard S1, that's why it's different"
Ops: "When did we change production?"
Developer: "I don't know, someone clicked in the portal"

3 months later:
"We need to recreate production in a new region"
"...we don't know how it's configured"
```

### The Secret Sprawl

```
Secrets scattered across:
├── appsettings.json (committed to Git)
├── Azure Portal → App Settings
├── Developer laptops
├── Slack messages ("here's the API key")
├── Confluence page (last updated 2 years ago)
└── Someone's memory

Result:
- "Which is the current password?"
- "Did we rotate that key?"
- "Who has access to this secret?"
```

### The Migration Disaster

```
Friday 4 PM: Deploy with database migration
Friday 4:15 PM: Migration fails halfway
Friday 4:30 PM: App crashes - schema mismatch
Friday 5:00 PM: Rollback? Data was inserted by new code
Friday 7:00 PM: Manual SQL fixes, data loss, weekend ruined

"Why didn't we test the migration?"
"Why didn't we have a rollback plan?"
```

---

## Azure Bicep Fundamentals

### Why Bicep?

```
ARM Template (JSON):
{
  "$schema": "https://schema.management.azure.com/...",
  "resources": [{
    "type": "Microsoft.Web/sites",
    "apiVersion": "2021-02-01",
    "name": "[parameters('siteName')]",
    "location": "[resourceGroup().location]",
    "properties": {
      "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('planName'))]"
    }
  }]
}

Bicep (same thing):
resource webApp 'Microsoft.Web/sites@2021-02-01' = {
  name: siteName
  location: resourceGroup().location
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```

### Project Structure

```
infra/
├── main.bicep                    # Entry point
├── parameters/
│   ├── dev.bicepparam           # Dev environment
│   ├── staging.bicepparam       # Staging environment
│   └── prod.bicepparam          # Production environment
├── modules/
│   ├── app-service.bicep        # App Service module
│   ├── container-app.bicep      # Container Apps module
│   ├── database.bicep           # PostgreSQL/SQL module
│   ├── key-vault.bicep          # Key Vault module
│   ├── monitoring.bicep         # Log Analytics, App Insights
│   └── networking.bicep         # VNet, private endpoints
└── scripts/
    ├── deploy.sh                # Deployment script
    └── validate.sh              # Validation script
```

---

## Complete OrderFlow Infrastructure

### Main Entry Point

```bicep
// infra/main.bicep
targetScope = 'subscription'

// ============================================
// Parameters
// ============================================
@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Azure region for resources')
param location string = 'eastus'

@description('Application name')
param appName string = 'orderflow'

@description('Container image tag')
param imageTag string = 'latest'

// Calculated values
var resourceGroupName = 'rg-${appName}-${environment}'
var tags = {
  Environment: environment
  Application: appName
  ManagedBy: 'Bicep'
  Repository: 'github.com/yourorg/orderflow'
}

// ============================================
// Resource Group
// ============================================
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: resourceGroupName
  location: location
  tags: tags
}

// ============================================
// Monitoring (deploy first - other resources depend on it)
// ============================================
module monitoring 'modules/monitoring.bicep' = {
  name: 'monitoring-${environment}'
  scope: rg
  params: {
    appName: appName
    environment: environment
    location: location
    tags: tags
  }
}

// ============================================
// Key Vault
// ============================================
module keyVault 'modules/key-vault.bicep' = {
  name: 'keyvault-${environment}'
  scope: rg
  params: {
    appName: appName
    environment: environment
    location: location
    tags: tags
    logAnalyticsWorkspaceId: monitoring.outputs.logAnalyticsWorkspaceId
  }
}

// ============================================
// Database
// ============================================
module database 'modules/database.bicep' = {
  name: 'database-${environment}'
  scope: rg
  params: {
    appName: appName
    environment: environment
    location: location
    tags: tags
    keyVaultName: keyVault.outputs.keyVaultName
    logAnalyticsWorkspaceId: monitoring.outputs.logAnalyticsWorkspaceId
  }
}

// ============================================
// Container App Environment
// ============================================
module containerApp 'modules/container-app.bicep' = {
  name: 'containerapp-${environment}'
  scope: rg
  params: {
    appName: appName
    environment: environment
    location: location
    tags: tags
    imageTag: imageTag
    keyVaultName: keyVault.outputs.keyVaultName
    databaseConnectionStringSecretUri: database.outputs.connectionStringSecretUri
    appInsightsConnectionString: monitoring.outputs.appInsightsConnectionString
    logAnalyticsWorkspaceId: monitoring.outputs.logAnalyticsWorkspaceId
  }
}

// ============================================
// Outputs
// ============================================
output resourceGroupName string = rg.name
output apiUrl string = containerApp.outputs.apiUrl
output keyVaultName string = keyVault.outputs.keyVaultName
output appInsightsName string = monitoring.outputs.appInsightsName
```

### Monitoring Module

```bicep
// infra/modules/monitoring.bicep
@description('Application name')
param appName string

@description('Environment')
param environment string

@description('Location')
param location string

@description('Tags')
param tags object

// ============================================
// Log Analytics Workspace
// ============================================
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'log-${appName}-${environment}'
  location: location
  tags: tags
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: environment == 'prod' ? 90 : 30
    features: {
      enableLogAccessUsingOnlyResourcePermissions: true
    }
  }
}

// ============================================
// Application Insights
// ============================================
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-${appName}-${environment}'
  location: location
  tags: tags
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
    IngestionMode: 'LogAnalytics'
    publicNetworkAccessForIngestion: 'Enabled'
    publicNetworkAccessForQuery: 'Enabled'
  }
}

// ============================================
// Alert Rules (Production only)
// ============================================
resource highErrorRateAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = if (environment == 'prod') {
  name: 'alert-high-error-rate-${appName}'
  location: 'global'
  tags: tags
  properties: {
    description: 'High error rate detected'
    severity: 1
    enabled: true
    scopes: [appInsights.id]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'HighErrorRate'
          metricName: 'requests/failed'
          operator: 'GreaterThan'
          threshold: 10
          timeAggregation: 'Count'
        }
      ]
    }
  }
}

// ============================================
// Outputs
// ============================================
output logAnalyticsWorkspaceId string = logAnalytics.id
output appInsightsId string = appInsights.id
output appInsightsName string = appInsights.name
output appInsightsConnectionString string = appInsights.properties.ConnectionString
output appInsightsInstrumentationKey string = appInsights.properties.InstrumentationKey
```

### Key Vault Module

```bicep
// infra/modules/key-vault.bicep
@description('Application name')
param appName string

@description('Environment')
param environment string

@description('Location')
param location string

@description('Tags')
param tags object

@description('Log Analytics Workspace ID')
param logAnalyticsWorkspaceId string

// ============================================
// Key Vault
// ============================================
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-${appName}-${environment}'
  location: location
  tags: tags
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true  // Use RBAC instead of access policies
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: environment == 'prod'
    publicNetworkAccess: 'Enabled'  // Restrict in production
    networkAcls: {
      defaultAction: 'Allow'
      bypass: 'AzureServices'
    }
  }
}

// ============================================
// Diagnostic Settings
// ============================================
resource diagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'diag-${keyVault.name}'
  scope: keyVault
  properties: {
    workspaceId: logAnalyticsWorkspaceId
    logs: [
      {
        categoryGroup: 'audit'
        enabled: true
      }
      {
        categoryGroup: 'allLogs'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
      }
    ]
  }
}

// ============================================
// Outputs
// ============================================
output keyVaultId string = keyVault.id
output keyVaultName string = keyVault.name
output keyVaultUri string = keyVault.properties.vaultUri
```

### Database Module

```bicep
// infra/modules/database.bicep
@description('Application name')
param appName string

@description('Environment')
param environment string

@description('Location')
param location string

@description('Tags')
param tags object

@description('Key Vault name for storing secrets')
param keyVaultName string

@description('Log Analytics Workspace ID')
param logAnalyticsWorkspaceId string

// ============================================
// PostgreSQL Flexible Server
// ============================================
var serverName = 'psql-${appName}-${environment}'
var databaseName = 'orderflow'

// Generate password (in production, use Key Vault reference)
var administratorLogin = 'orderflowadmin'
@secure()
param administratorLoginPassword string = newGuid()

resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2023-03-01-preview' = {
  name: serverName
  location: location
  tags: tags
  sku: {
    name: environment == 'prod' ? 'Standard_D4ds_v4' : 'Standard_B1ms'
    tier: environment == 'prod' ? 'GeneralPurpose' : 'Burstable'
  }
  properties: {
    version: '16'
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorLoginPassword
    storage: {
      storageSizeGB: environment == 'prod' ? 128 : 32
    }
    backup: {
      backupRetentionDays: environment == 'prod' ? 35 : 7
      geoRedundantBackup: environment == 'prod' ? 'Enabled' : 'Disabled'
    }
    highAvailability: {
      mode: environment == 'prod' ? 'ZoneRedundant' : 'Disabled'
    }
  }
}

// Database
resource database 'Microsoft.DBforPostgreSQL/flexibleServers/databases@2023-03-01-preview' = {
  parent: postgresServer
  name: databaseName
  properties: {
    charset: 'UTF8'
    collation: 'en_US.utf8'
  }
}

// Firewall rule for Azure services
resource allowAzureServices 'Microsoft.DBforPostgreSQL/flexibleServers/firewallRules@2023-03-01-preview' = {
  parent: postgresServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}

// ============================================
// Store Connection String in Key Vault
// ============================================
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: keyVaultName
}

var connectionString = 'Host=${postgresServer.properties.fullyQualifiedDomainName};Database=${databaseName};Username=${administratorLogin};Password=${administratorLoginPassword};SSL Mode=Require;Trust Server Certificate=true'

resource connectionStringSecret 'Microsoft.KeyVault/vaults/secrets@2023-07-01' = {
  parent: keyVault
  name: 'database-connection-string'
  properties: {
    value: connectionString
    contentType: 'text/plain'
    attributes: {
      enabled: true
    }
  }
}

// ============================================
// Outputs
// ============================================
output serverName string = postgresServer.name
output serverFqdn string = postgresServer.properties.fullyQualifiedDomainName
output databaseName string = database.name
output connectionStringSecretUri string = connectionStringSecret.properties.secretUri
```

### Container App Module

```bicep
// infra/modules/container-app.bicep
@description('Application name')
param appName string

@description('Environment')
param environment string

@description('Location')
param location string

@description('Tags')
param tags object

@description('Container image tag')
param imageTag string

@description('Key Vault name')
param keyVaultName string

@description('Database connection string secret URI')
param databaseConnectionStringSecretUri string

@description('App Insights connection string')
param appInsightsConnectionString string

@description('Log Analytics Workspace ID')
param logAnalyticsWorkspaceId string

// ============================================
// Container App Environment
// ============================================
resource containerAppEnv 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: 'cae-${appName}-${environment}'
  location: location
  tags: tags
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: reference(logAnalyticsWorkspaceId, '2022-10-01').customerId
        sharedKey: listKeys(logAnalyticsWorkspaceId, '2022-10-01').primarySharedKey
      }
    }
    zoneRedundant: environment == 'prod'
  }
}

// ============================================
// User Assigned Managed Identity
// ============================================
resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-${appName}-${environment}'
  location: location
  tags: tags
}

// Grant Key Vault access to managed identity
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' existing = {
  name: keyVaultName
}

resource keyVaultRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, managedIdentity.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: managedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

// ============================================
// Container App - API
// ============================================
resource apiContainerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'ca-${appName}-api-${environment}'
  location: location
  tags: tags
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${managedIdentity.id}': {}
    }
  }
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      activeRevisionsMode: 'Multiple'  // Enable traffic splitting
      ingress: {
        external: true
        targetPort: 8080
        transport: 'http'
        traffic: [
          {
            latestRevision: true
            weight: 100
          }
        ]
        corsPolicy: {
          allowedOrigins: environment == 'prod'
            ? ['https://app.orderflow.example.com']
            : ['*']
          allowedMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
          allowedHeaders: ['*']
        }
      }
      secrets: [
        {
          name: 'database-connection-string'
          keyVaultUrl: databaseConnectionStringSecretUri
          identity: managedIdentity.id
        }
      ]
      registries: [
        {
          server: 'ghcr.io'
          identity: managedIdentity.id
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'api'
          image: 'ghcr.io/yourorg/orderflow-api:${imageTag}'
          resources: {
            cpu: environment == 'prod' ? json('1.0') : json('0.5')
            memory: environment == 'prod' ? '2Gi' : '1Gi'
          }
          env: [
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: environment == 'prod' ? 'Production' : 'Development'
            }
            {
              name: 'ConnectionStrings__Database'
              secretRef: 'database-connection-string'
            }
            {
              name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
              value: appInsightsConnectionString
            }
          ]
          probes: [
            {
              type: 'Liveness'
              httpGet: {
                path: '/health/live'
                port: 8080
              }
              initialDelaySeconds: 10
              periodSeconds: 30
            }
            {
              type: 'Readiness'
              httpGet: {
                path: '/health/ready'
                port: 8080
              }
              initialDelaySeconds: 5
              periodSeconds: 10
            }
          ]
        }
      ]
      scale: {
        minReplicas: environment == 'prod' ? 2 : 0
        maxReplicas: environment == 'prod' ? 10 : 3
        rules: [
          {
            name: 'http-scaling'
            http: {
              metadata: {
                concurrentRequests: '100'
              }
            }
          }
        ]
      }
    }
  }
}

// ============================================
// Outputs
// ============================================
output apiUrl string = 'https://${apiContainerApp.properties.configuration.ingress.fqdn}'
output containerAppName string = apiContainerApp.name
output managedIdentityId string = managedIdentity.id
```

---

## Parameter Files

### Development Environment

```bicep
// infra/parameters/dev.bicepparam
using '../main.bicep'

param environment = 'dev'
param location = 'eastus'
param appName = 'orderflow'
param imageTag = 'develop'
```

### Production Environment

```bicep
// infra/parameters/prod.bicepparam
using '../main.bicep'

param environment = 'prod'
param location = 'eastus'
param appName = 'orderflow'
param imageTag = 'v1.0.0'  // Explicit version for production
```

---

## Database Migrations

### EF Core Migration Strategy

```csharp
// Safe migration approach - separate migration from deployment

// 1. Migration Console App
public class Program
{
    public static async Task Main(string[] args)
    {
        var builder = Host.CreateApplicationBuilder(args);

        builder.Services.AddDbContext<OrderFlowDbContext>(options =>
            options.UseNpgsql(builder.Configuration.GetConnectionString("Database")));

        var app = builder.Build();

        using var scope = app.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();
        var logger = scope.ServiceProvider.GetRequiredService<ILogger<Program>>();

        try
        {
            logger.LogInformation("Starting database migration...");

            var pendingMigrations = await db.Database.GetPendingMigrationsAsync();
            if (pendingMigrations.Any())
            {
                logger.LogInformation("Applying {Count} pending migrations: {Migrations}",
                    pendingMigrations.Count(), string.Join(", ", pendingMigrations));

                await db.Database.MigrateAsync();
                logger.LogInformation("Migration completed successfully");
            }
            else
            {
                logger.LogInformation("No pending migrations");
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Migration failed");
            Environment.Exit(1);
        }
    }
}
```

### Migration in CI/CD

```yaml
# Migration job in CD pipeline
jobs:
  migrate-database:
    name: Run Database Migrations
    runs-on: ubuntu-latest
    environment: staging  # Requires approval for prod

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get database connection string
        id: db
        run: |
          CONNECTION=$(az keyvault secret show \
            --vault-name ${{ vars.KEY_VAULT_NAME }} \
            --name database-connection-string \
            --query value -o tsv)
          echo "::add-mask::$CONNECTION"
          echo "connection=$CONNECTION" >> $GITHUB_OUTPUT

      - name: Create database backup
        run: |
          # Create backup before migration (production only)
          if [ "${{ vars.ENVIRONMENT }}" == "prod" ]; then
            az postgres flexible-server backup create \
              --resource-group ${{ vars.RESOURCE_GROUP }} \
              --server-name ${{ vars.DATABASE_SERVER }} \
              --backup-name "pre-migration-$(date +%Y%m%d_%H%M%S)"
          fi

      - name: Run migrations
        run: |
          dotnet ef database update \
            --project src/OrderFlow.Infrastructure \
            --startup-project src/OrderFlow.Api \
            --connection "${{ steps.db.outputs.connection }}"

      - name: Verify migration
        run: |
          dotnet ef migrations list \
            --project src/OrderFlow.Infrastructure \
            --startup-project src/OrderFlow.Api \
            --connection "${{ steps.db.outputs.connection }}" \
            --no-build
```

### Migration Best Practices

```sql
-- Safe migration patterns

-- 1. Add column with default (no table lock)
ALTER TABLE orders ADD COLUMN created_by VARCHAR(255) DEFAULT 'system';

-- 2. Add index concurrently (PostgreSQL)
CREATE INDEX CONCURRENTLY idx_orders_customer_id ON orders(customer_id);

-- 3. Rename column in stages
-- Stage 1: Add new column
ALTER TABLE orders ADD COLUMN customer_email VARCHAR(255);
-- Stage 2: Copy data (in batches)
UPDATE orders SET customer_email = email WHERE customer_email IS NULL LIMIT 10000;
-- Stage 3: Update application to use new column
-- Stage 4: Drop old column (after confirming no usage)
ALTER TABLE orders DROP COLUMN email;

-- AVOID: Operations that lock tables
-- ALTER TABLE orders ADD COLUMN new_col VARCHAR NOT NULL;  -- Locks table!
-- CREATE INDEX idx_orders_date ON orders(order_date);      -- Locks table in some DBs
```

---

## Secrets Management

### Key Vault Integration in .NET

```csharp
// Program.cs - Load secrets from Key Vault
var builder = WebApplication.CreateBuilder(args);

// Add Key Vault configuration
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUri = new Uri(builder.Configuration["KeyVault:Uri"]!);
    builder.Configuration.AddAzureKeyVault(
        keyVaultUri,
        new DefaultAzureCredential());
}

// Secrets are now available via Configuration
// builder.Configuration["database-connection-string"]
```

### Secret Rotation Runbook

```markdown
# Secret Rotation Runbook: Database Password

## Prerequisites
- Azure CLI authenticated
- Access to Key Vault and PostgreSQL

## Steps

### 1. Generate New Password
```bash
NEW_PASSWORD=$(openssl rand -base64 32)
```

### 2. Update PostgreSQL Password
```bash
az postgres flexible-server update \
  --resource-group orderflow-prod-rg \
  --name psql-orderflow-prod \
  --admin-password "$NEW_PASSWORD"
```

### 3. Update Key Vault Secret
```bash
# Get current connection string
CURRENT=$(az keyvault secret show \
  --vault-name kv-orderflow-prod \
  --name database-connection-string \
  --query value -o tsv)

# Replace password in connection string
NEW_CONNECTION=$(echo $CURRENT | sed "s/Password=[^;]*/Password=$NEW_PASSWORD/")

# Update secret
az keyvault secret set \
  --vault-name kv-orderflow-prod \
  --name database-connection-string \
  --value "$NEW_CONNECTION"
```

### 4. Restart Application
```bash
az containerapp revision restart \
  --name ca-orderflow-api-prod \
  --resource-group orderflow-prod-rg
```

### 5. Verify
```bash
# Check application health
curl -f https://api.orderflow.example.com/health/ready

# Verify database connectivity in logs
az containerapp logs show \
  --name ca-orderflow-api-prod \
  --resource-group orderflow-prod-rg \
  --tail 100 | grep -i "database"
```

## Rollback
If issues occur:
1. Revert Key Vault secret to previous version
2. Restart application
3. Investigate and retry
```

### Automated Secret Rotation

```yaml
# .github/workflows/rotate-secrets.yml
name: Rotate Secrets

on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly
  workflow_dispatch:
    inputs:
      secret_name:
        description: 'Secret to rotate'
        required: true
        type: choice
        options:
          - database-password
          - api-key
          - jwt-secret

jobs:
  rotate:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Generate new secret
        id: generate
        run: |
          NEW_SECRET=$(openssl rand -base64 32)
          echo "::add-mask::$NEW_SECRET"
          echo "value=$NEW_SECRET" >> $GITHUB_OUTPUT

      - name: Update Key Vault
        run: |
          az keyvault secret set \
            --vault-name ${{ vars.KEY_VAULT_NAME }} \
            --name ${{ inputs.secret_name }} \
            --value "${{ steps.generate.outputs.value }}"

      - name: Restart application
        run: |
          az containerapp revision restart \
            --name ${{ vars.CONTAINER_APP_NAME }} \
            --resource-group ${{ vars.RESOURCE_GROUP }}

      - name: Verify deployment
        run: |
          sleep 60
          curl -f https://${{ vars.API_URL }}/health/ready

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Secret rotation failed for ${{ inputs.secret_name }}!"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## CI/CD Integration

### Infrastructure Deployment Pipeline

```yaml
# .github/workflows/infrastructure.yml
name: Infrastructure Deployment

on:
  push:
    branches: [main]
    paths:
      - 'infra/**'
  pull_request:
    branches: [main]
    paths:
      - 'infra/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  # ============================================
  # VALIDATE
  # ============================================
  validate:
    name: Validate Bicep
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Validate Bicep
        run: |
          az bicep build --file infra/main.bicep

      - name: Run Bicep linter
        run: |
          az bicep lint --file infra/main.bicep

  # ============================================
  # PREVIEW (What-If)
  # ============================================
  preview:
    name: Preview Changes
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: What-If Analysis
        id: whatif
        run: |
          az deployment sub what-if \
            --location eastus \
            --template-file infra/main.bicep \
            --parameters infra/parameters/dev.bicepparam \
            --no-pretty-print > whatif.json

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const whatif = fs.readFileSync('whatif.json', 'utf8');

            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Infrastructure Changes Preview\n\`\`\`json\n${whatif}\n\`\`\``
            });

  # ============================================
  # DEPLOY
  # ============================================
  deploy:
    name: Deploy Infrastructure
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name != 'pull_request'
    environment: ${{ github.event.inputs.environment || 'dev' }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy Infrastructure
        id: deploy
        run: |
          ENV="${{ github.event.inputs.environment || 'dev' }}"

          az deployment sub create \
            --location eastus \
            --template-file infra/main.bicep \
            --parameters infra/parameters/${ENV}.bicepparam \
            --name "orderflow-${ENV}-$(date +%Y%m%d-%H%M%S)" \
            --query properties.outputs -o json > outputs.json

          echo "API URL: $(jq -r '.apiUrl.value' outputs.json)"

      - name: Upload outputs
        uses: actions/upload-artifact@v4
        with:
          name: deployment-outputs
          path: outputs.json
```

---

## Terraform Alternative

### Equivalent Terraform Configuration

```hcl
# infra/terraform/main.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateorderflow"
    container_name       = "tfstate"
    key                  = "orderflow.tfstate"
  }
}

provider "azurerm" {
  features {}
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "location" {
  type    = string
  default = "eastus"
}

variable "app_name" {
  type    = string
  default = "orderflow"
}

locals {
  resource_group_name = "rg-${var.app_name}-${var.environment}"
  tags = {
    Environment = var.environment
    Application = var.app_name
    ManagedBy   = "Terraform"
  }
}

resource "azurerm_resource_group" "main" {
  name     = local.resource_group_name
  location = var.location
  tags     = local.tags
}

module "monitoring" {
  source = "./modules/monitoring"

  resource_group_name = azurerm_resource_group.main.name
  location            = var.location
  app_name            = var.app_name
  environment         = var.environment
  tags                = local.tags
}

module "key_vault" {
  source = "./modules/key-vault"

  resource_group_name        = azurerm_resource_group.main.name
  location                   = var.location
  app_name                   = var.app_name
  environment                = var.environment
  tags                       = local.tags
  log_analytics_workspace_id = module.monitoring.log_analytics_workspace_id
}

# ... additional modules
```

---

## Hands-On Exercise: Deploy OrderFlow Infrastructure

### Step 1: Create Bicep Modules

Create the modular infrastructure:
- `main.bicep` entry point
- `monitoring.bicep` module
- `key-vault.bicep` module
- `database.bicep` module
- `container-app.bicep` module

### Step 2: Create Parameter Files

Create environment-specific parameters:
- `dev.bicepparam`
- `staging.bicepparam`
- `prod.bicepparam`

### Step 3: Deploy to Dev

```bash
# Validate
az bicep build --file infra/main.bicep

# What-if
az deployment sub what-if \
  --location eastus \
  --template-file infra/main.bicep \
  --parameters infra/parameters/dev.bicepparam

# Deploy
az deployment sub create \
  --location eastus \
  --template-file infra/main.bicep \
  --parameters infra/parameters/dev.bicepparam
```

### Step 4: Set Up CI/CD Pipeline

Create `.github/workflows/infrastructure.yml`:
- Validate on PR
- What-if preview in PR comments
- Deploy on merge to main

### Step 5: Configure Secrets

Store deployment credentials:
- Create Azure Service Principal
- Configure OIDC authentication
- Store credentials in GitHub Secrets

---

## Deliverables

1. **Bicep Modules**: Complete, modular infrastructure code
2. **Parameter Files**: Environment-specific configurations
3. **CI/CD Pipeline**: Automated infrastructure deployment
4. **Migration Script**: EF Core migration automation
5. **Secret Rotation Runbook**: Documented rotation procedures
6. **Documentation**: README with deployment instructions

---

## Reflection Questions

1. Why use subscription-level deployment instead of resource group level?
2. What's the risk of storing Terraform state locally?
3. How do you handle breaking changes in database migrations?
4. What's the difference between soft delete and purge protection in Key Vault?

---

## Resources

### Azure Bicep
- [Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Bicep Playground](https://aka.ms/bicepdemo)
- [Bicep Best Practices](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/best-practices)
- [Azure Bicep Deep Dive](https://www.youtube.com/watch?v=yFw03RGdxFc) - John Savill

### Terraform
- [Terraform Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Terraform Best Practices](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices)
- [HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)

### Database Migrations
- [EF Core Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Flyway](https://flywaydb.org/documentation/)
- [Safe Database Migrations](https://www.youtube.com/watch?v=5DHRhSbDMQI)

### Secrets Management
- [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/)
- [Key Vault with Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/manage-secrets)
- [Secret Rotation](https://learn.microsoft.com/en-us/azure/key-vault/secrets/tutorial-rotation)

---

**Module Complete!** Return to [Module 11 Overview](./README.md) or continue to [Module 12 — Professional Development](../12-professional/README.md)
