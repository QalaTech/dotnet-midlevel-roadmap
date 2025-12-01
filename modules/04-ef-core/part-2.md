# 04 — Entity Framework Core & Data Access

## Part 2 / 5 — Migrations and Schema Evolution

---

## Why This Matters

Databases live in production. Changing them is dangerous:
- Data loss is permanent
- Downtime costs money
- Rollbacks are hard
- Mistakes affect everyone

Migrations are your safety net — when done right.

---

## Core Concept #5 — Migration Fundamentals

### Why This Matters

Migrations are version control for your database. They:
- Document every schema change
- Make changes repeatable across environments
- Enable rollbacks (sometimes)
- Coordinate changes across teams

### What Goes Wrong Without This

**Real scenario:** Two developers both add columns to the same table. They merge their code. Nobody runs migrations in the correct order. Production has columns in a different order than staging. Queries start failing because of column ordinal assumptions.

**Real scenario:** Developer runs `dotnet ef database update` on production directly from their laptop. The connection string points to prod because of leftover environment variables. Migration intended for dev wipes production data.

### Basic Migration Workflow

```bash
# Create a migration
dotnet ef migrations add AddCustomerCreditLimit \
    --project src/OrderFlow.Infrastructure \
    --startup-project src/OrderFlow.Api

# Review what it will do
dotnet ef migrations script \
    --project src/OrderFlow.Infrastructure \
    --startup-project src/OrderFlow.Api

# Apply to local database
dotnet ef database update \
    --project src/OrderFlow.Infrastructure \
    --startup-project src/OrderFlow.Api

# Generate SQL script for DBA review (production)
dotnet ef migrations script FromMigration ToMigration \
    --idempotent \
    --output migration.sql
```

### Migration File Anatomy

```csharp
// 20240115120000_AddCustomerCreditLimit.cs
public partial class AddCustomerCreditLimit : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<decimal>(
            name: "CreditLimit",
            table: "Customers",
            type: "decimal(18,2)",
            nullable: false,
            defaultValue: 0m);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(
            name: "CreditLimit",
            table: "Customers");
    }
}
```

### The ModelSnapshot

```csharp
// OrderFlowDbContextModelSnapshot.cs
// THIS IS CRITICAL - represents current expected state
// Never edit manually except in emergencies
// If corrupted, migrations will fail or generate wrong SQL
```

---

## Core Concept #6 — Safe Migration Patterns

### Why This Matters

Some schema changes are destructive. Doing them wrong means data loss or downtime.

### What Goes Wrong Without This

**War story #1 — The Column Rename Disaster:**
Developer renames `CustomerName` to `Name` in C#. EF Core generates:
```sql
ALTER TABLE Customers DROP COLUMN CustomerName;
ALTER TABLE Customers ADD Name nvarchar(100);
```
All customer names are gone. Permanently.

**War story #2 — The NOT NULL Catastrophe:**
Developer adds a new required column without a default. Migration runs on production:
```sql
ALTER TABLE Orders ADD ShippingMethod nvarchar(50) NOT NULL;
```
Error: Cannot insert NULL into column. 50,000 existing orders prevent migration.
Deployment fails at 2 AM.

**War story #3 — The Foreign Key Lock:**
Developer adds a foreign key to a large table. Migration takes an exclusive lock:
```sql
ALTER TABLE OrderItems ADD CONSTRAINT FK_OrderItems_Products
    FOREIGN KEY (ProductId) REFERENCES Products(Id);
```
All writes to OrderItems are blocked. Site goes down for 15 minutes while 10 million rows are validated.

### Safe Pattern: Column Rename (Expand-Contract)

```csharp
// WRONG: Single migration that loses data
migrationBuilder.RenameColumn("CustomerName", "Customers", "Name");
// EF Core might generate DROP + ADD instead of RENAME

// RIGHT: Three-phase approach

// Phase 1: Add new column (deploy with old code)
public partial class AddNameColumn : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "Name",
            table: "Customers",
            type: "nvarchar(200)",
            nullable: true);

        // Copy existing data
        migrationBuilder.Sql(
            "UPDATE Customers SET Name = CustomerName");
    }
}

// Phase 2: Update code to write to both columns, read from new
// Deploy this code change

// Phase 3: Make new column required, drop old column
public partial class RemoveCustomerNameColumn : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AlterColumn<string>(
            name: "Name",
            table: "Customers",
            type: "nvarchar(200)",
            nullable: false,
            oldNullable: true);

        migrationBuilder.DropColumn(
            name: "CustomerName",
            table: "Customers");
    }
}
```

### Safe Pattern: Add Required Column

```csharp
// WRONG: Add NOT NULL without default
migrationBuilder.AddColumn<string>(
    name: "ShippingMethod",
    table: "Orders",
    nullable: false);  // Fails if table has data

// RIGHT: Add nullable, backfill, then make required
public partial class AddShippingMethod : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Step 1: Add nullable column
        migrationBuilder.AddColumn<string>(
            name: "ShippingMethod",
            table: "Orders",
            type: "nvarchar(50)",
            nullable: true);

        // Step 2: Backfill existing data
        migrationBuilder.Sql(
            "UPDATE Orders SET ShippingMethod = 'Standard' WHERE ShippingMethod IS NULL");

        // Step 3: Make NOT NULL
        migrationBuilder.AlterColumn<string>(
            name: "ShippingMethod",
            table: "Orders",
            type: "nvarchar(50)",
            nullable: false,
            oldNullable: true);
    }
}
```

### Safe Pattern: Add Foreign Key to Large Table

```csharp
// WRONG: Add FK constraint directly (locks table)
migrationBuilder.AddForeignKey(...);

// RIGHT: Add FK with NOCHECK, validate separately
public partial class AddOrderItemsProductFK : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Add constraint without validating existing data
        migrationBuilder.Sql(@"
            ALTER TABLE OrderItems WITH NOCHECK
            ADD CONSTRAINT FK_OrderItems_Products
            FOREIGN KEY (ProductId) REFERENCES Products(Id)");

        // Validate in batches during low-traffic period
        // (done manually or in separate migration)
    }
}

// Later: Validate constraint
// ALTER TABLE OrderItems WITH CHECK CHECK CONSTRAINT FK_OrderItems_Products
```

### Safe Pattern: Change Column Type

```csharp
// Changing int to bigint, or varchar(50) to varchar(200)
public partial class ExpandOrderNumberLength : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Usually safe - expanding length
        migrationBuilder.AlterColumn<string>(
            name: "OrderNumber",
            table: "Orders",
            type: "nvarchar(100)",  // Was nvarchar(50)
            nullable: false);
    }
}

// Changing string to int, or shrinking size - DANGEROUS
// Requires expand-contract pattern with data migration
```

---

## Core Concept #7 — Migration Deployment Strategies

### Why This Matters

How you deploy migrations affects:
- Risk of data loss
- Downtime requirements
- Rollback capability
- Team coordination

### Strategy 1: Application Startup Migrations

```csharp
// Program.cs
using var scope = app.Services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<OrderFlowDbContext>();
await context.Database.MigrateAsync();

// Pros:
// - Simple
// - Migrations always current
// Cons:
// - App can't start if migration fails
// - Multiple instances race to migrate
// - Can't preview migration before running
// - Bad for zero-downtime deployments
```

**When to use:** Small apps, development, where downtime is acceptable.

### Strategy 2: CI/CD Pipeline Migrations

```yaml
# deploy-pipeline.yml
stages:
  - stage: ApplyMigrations
    jobs:
      - job: Migrate
        steps:
          - script: |
              dotnet ef migrations script \
                --idempotent \
                --output $(Build.ArtifactStagingDirectory)/migration.sql
            displayName: 'Generate migration script'

          # Manual approval gate here for production
          - task: SqlAzureDacpacDeployment@1
            inputs:
              azureSubscription: 'Production'
              SqlFile: '$(Build.ArtifactStagingDirectory)/migration.sql'

  - stage: DeployApp
    dependsOn: ApplyMigrations
    # Deploy app only after migrations succeed
```

**When to use:** Production systems, when you need review and approval.

### Strategy 3: Blue-Green with Migrations

```
┌─────────────────────────────────────────────────────────┐
│                     Load Balancer                        │
└────────────────┬───────────────────────┬────────────────┘
                 │                       │
         ┌───────▼───────┐       ┌───────▼───────┐
         │   Blue (v1)   │       │  Green (v2)   │
         │   Old code    │       │  New code     │
         └───────┬───────┘       └───────┬───────┘
                 │                       │
         ┌───────▼───────────────────────▼───────┐
         │            Shared Database             │
         │  (Must support both v1 and v2 code)   │
         └───────────────────────────────────────┘
```

**Requirement:** Migrations must be backward compatible!

```csharp
// This migration breaks blue-green deployment:
migrationBuilder.DropColumn("OldColumn");  // v1 code still reads it!

// Safe migration for blue-green:
// 1. Add new column (both versions work)
// 2. Deploy green (writes to both columns)
// 3. Switch traffic to green
// 4. Later migration: remove old column
```

---

## Core Concept #8 — Rollback Strategies

### Why This Matters

Migrations fail. Having a rollback plan is essential.

### What Goes Wrong Without This

**War story — The Friday Deploy:**
Team deploys new migration Friday at 5 PM. Migration fails halfway through. No rollback plan exists. Database is in inconsistent state. Team spends all weekend fixing it manually.

### Rollback Option 1: Down Migrations

```csharp
protected override void Down(MigrationBuilder migrationBuilder)
{
    // Reverses the Up migration
    migrationBuilder.DropColumn("NewColumn", "Orders");
}

// Run: dotnet ef database update PreviousMigrationName
```

**Limitations:**
- Data in new columns is lost
- Can't rollback data transformations
- Doesn't work for all changes

### Rollback Option 2: Database Snapshots

```sql
-- Before migration
CREATE DATABASE OrderFlow_Snapshot ON
(
    NAME = OrderFlow,
    FILENAME = 'C:\Snapshots\OrderFlow_20240115.ss'
) AS SNAPSHOT OF OrderFlow;

-- If migration fails
RESTORE DATABASE OrderFlow FROM DATABASE_SNAPSHOT = 'OrderFlow_Snapshot';
```

### Rollback Option 3: Point-in-Time Recovery

```bash
# Azure SQL
az sql db restore \
    --dest-name OrderFlow_Restored \
    --name OrderFlow \
    --resource-group mygroup \
    --server myserver \
    --time "2024-01-15T14:00:00Z"
```

### Rollback Option 4: Forward-Fix

Sometimes the fastest fix is another migration:

```csharp
// Original migration broke something
// Instead of rollback, deploy a fix-forward migration:
public partial class FixBrokenMigration : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        // Fix the problem caused by previous migration
        migrationBuilder.Sql(
            "UPDATE Orders SET Status = 'Pending' WHERE Status IS NULL");
    }
}
```

---

## Core Concept #9 — Team Workflows

### Why This Matters

Multiple developers creating migrations causes conflicts.

### What Goes Wrong Without This

**Real scenario:** Two developers create migrations from the same snapshot. Both modify the same table. Migrations have conflicting changes. Which one wins? Neither — the merge fails.

### Workflow: Trunk-Based Development

```
main branch
    │
    ├── Developer A: Create feature + migration
    │   └── PR: Review migration SQL
    │
    ├── Developer B: Wait for A's PR to merge
    │   └── Pull main, create migration from updated snapshot
    │
    └── Single migration per change, always sequential
```

### Workflow: Resolving Migration Conflicts

```bash
# Scenario: You have local migration, main has different migration

# 1. Remove your local migration
dotnet ef migrations remove

# 2. Pull and update to latest
git pull origin main
dotnet ef database update

# 3. Recreate your migration against updated snapshot
dotnet ef migrations add YourMigration
```

### Workflow: Squashing Development Migrations

```bash
# Development has 15 small migrations
# For production, squash into single migration

# 1. Create migration script from all dev migrations
dotnet ef migrations script InitialCreate CurrentMigration \
    --idempotent --output combined.sql

# 2. Reset to initial state (dev only)
dotnet ef database update InitialCreate
dotnet ef migrations remove  # Repeat for all migrations

# 3. Create single combined migration
dotnet ef migrations add CombinedFeatureX
```

---

## Hands-On: OrderFlow Migration Strategy

### Task 1 — Create Migration Plan

Document the migration strategy for OrderFlow:
- When are migrations applied?
- Who approves production migrations?
- What's the rollback procedure?

### Task 2 — Practice Expand-Contract

Implement a column rename using the expand-contract pattern:
1. Add new column with migration
2. Write code that uses both
3. Deploy to staging
4. Remove old column with migration

### Task 3 — Create Rollback Procedure

Create a documented rollback procedure:
- How to rollback to previous migration
- How to restore from backup
- Communication plan if rollback needed

### Task 4 — Test Migration Failure

1. Create a migration that will fail (invalid SQL)
2. Observe what happens
3. Practice recovery

---

## Deliverables

1. **Migration strategy document** with approval workflow
2. **Expand-contract example** demonstrating safe column rename
3. **Rollback procedure** document
4. **CI/CD pipeline** that generates migration scripts for review

---

## Resources

### Must-Read
- [EF Core Migrations](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/)
- [Martin Fowler: Evolutionary Database Design](https://martinfowler.com/articles/evodb.html)
- [Expand-Contract Pattern](https://www.prisma.io/dataguide/types/relational/expand-and-contract-pattern)

### Videos
- [Nick Chapsas: EF Core Migrations in Production](https://www.youtube.com/watch?v=g7mRLnfbKTY)
- [Raw Coding: Database Migrations Strategy](https://www.youtube.com/watch?v=yQzJrK3XBg8)

### Tools
- [Flyway](https://flywaydb.org/) — Database migration tool
- [DbUp](https://dbup.github.io/) — .NET migration library
- [EF Core Power Tools](https://marketplace.visualstudio.com/items?itemName=ErikEJ.EFCorePowerTools)

---

## Reflection Questions

1. Why can't you always use the Down() method to rollback?
2. What makes a migration "backward compatible"?
3. When would you use idempotent scripts vs standard migrations?
4. How do you handle migrations in a microservices architecture?

---

**Next:** [Part 3 — Querying and Performance](./part-3.md)
