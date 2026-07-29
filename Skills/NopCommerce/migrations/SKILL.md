---
name: migrations
description: >-
  Write nopCommerce plugin schema migrations with FluentMigrator, choosing
  between Migration, AutoReversingMigration, and ForwardOnlyMigration and the
  correct MigrationProcessType (Installation/Update/NoMatter). Use when
  creating a new table, altering columns on an existing table, or making any
  other schema change for a plugin — always ask the user for the migration
  type and process type, and bump the plugin.json version afterward.
---
# Basic Structure of a schema migration in nopCommerce

## When to Use
Activate this skill when creating a new schema migration for a plugin.

## How to Use
> **Always ask the user for the migration type & migration process type**
- Common migration types: `Migration`, `AutoReversingMigration`
- Common migration process types: `MigrationProcessType.Installation`, `MigrationProcessType.Update`, `MigrationProcessType.NoMatter`
- Use `NopSchemaMigration` attribute for migration classes
- Use `ForwardOnlyMigration` for migrations that do not require a down method. For instance when adding new columns to an existing table.
- Always bump plugin version in `plugin.json` after creating a new schema migration

### Template for SchemaMigration
```csharp
[NopSchemaMigration("2024/01/20 12:00:00:0000000", "Plugin.Base schema migration", MigrationProcessType.Installation)]
public class SchemaMigration : ForwardOnlyMigration
{
    /// <summary>
    /// Resolves the table name from BaseNameCompatibility via NameCompatibilityManager
    /// </summary>
    public static string TableName<T>() where T : BaseEntity
    {
        return NameCompatibilityManager.GetTableName(typeof(T));
    }

    public override void Up()
    {
        if (!Schema.Table(TableName<MyEntity>()).Exists())
        {
            Create.TableFor<MyEntity>();
        }
    }
}
```

```csharp
[NopSchemaMigration("2024/01/20 12:00:00:0000000", "Plugin.Base schema migration", MigrationProcessType.Installation)]
public class SchemaMigration : AutoReversingMigration
{
    public override void Up()
    {
      Create.TableFor<MyEntity>();
    }
}
```

```C#
[NopSchemaMigration("2025/01/15 00:00:00", "Plugin: Create initial schema", MigrationProcessType.NoMatter)]
public class SchemaMigration : Migration
{
    /// <summary>
    /// Resolves the table name from BaseNameCompatibility via NameCompatibilityManager
    /// </summary>
    public static string TableName<T>() where T : BaseEntity
    {
        return NameCompatibilityManager.GetTableName(typeof(T));
    }

    public override void Up()
    {
        if (!Schema.Table(TableName<MyEntity>()).Exists())
        {
            Create.TableFor<MyEntity>();
        }
    }

    public override void Down()
    {
        if (Schema.Table(TableName<MyEntity>()).Exists())
        {
            Delete.Table(TableName<MyEntity>());
        }
    }
}
```

**Columns migration example:**

```csharp
[NopSchemaMigration("2026-02-04 00:00:00", "Plugin: PaymentVault add IsLegacy and BillingAgreementId", MigrationProcessType.NoMatter)]
public class PaymentVault_AddIsLegacyColumnMigration : Migration
{
    public static string TableName<T>() where T : BaseEntity
    {
        return NameCompatibilityManager.GetTableName(typeof(T));
    }

    public override void Up()
    {
        var tableName = TableName<MyEntity>();

        if (Schema.Table(tableName).Exists() &&
            !Schema.Table(tableName).Column(nameof(MyEntity.Column1)).Exists())
        {
            Alter.Table(tableName)
                .AddColumn(nameof(MyEntity.Column1))
                .AsBoolean().NotNullable().WithDefaultValue(false);
        }

        if (Schema.Table(tableName).Exists() &&
            !Schema.Table(tableName).Column(nameof(MyEntity.Column2)).Exists())
        {
            Alter.Table(tableName)
                .AddColumn(nameof(MyEntity.Column2))
                .AsString().Nullable();
        }
    }

    public override void Down()
    {
        var tableName = TableName<MyEntity>();

        if (Schema.Table(tableName).Exists() &&
            Schema.Table(tableName).Column(nameof(MyEntity.Column1)).Exists())
        {
            Delete.Column(nameof(MyEntity.Column1)).FromTable(tableName);
        }

        if (Schema.Table(tableName).Exists() &&
            Schema.Table(tableName).Column(nameof(MyEntity.Column2)).Exists())
        {
            Delete.Column(nameof(MyEntity.Column2)).FromTable(tableName);
        }
    }
}
```

## Guardrails
- Use `[NopSchemaMigration]` (not `[NopMigration]`)
- `TableName<T>()` helper resolves names via `NameCompatibilityManager.GetTableName()` — this reads from `BaseNameCompatibility`
- Always check `Schema.Table(...).Exists()` for ```ForwardOnlyMigration` and `Migration` types before creating or deleting tables
