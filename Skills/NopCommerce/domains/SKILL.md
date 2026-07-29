---
name: domains
description: >-
  Create a nopCommerce plugin domain entity (BaseEntity subclass) along with
  its supporting data-layer files — BaseNameCompatibility table-name mapping
  and the FluentMigrator entity Builder. Use when adding a new database
  table/entity to a plugin; this always requires asking the user for the
  TablePrefix before generating anything.
---

# Basic Structure of a data layer in nopCommerce

## When to Use
Activate this skill when creating new database tables for a plugin and adding domain entities.

## How to Use

### Table Naming Conventions
- Use PascalCase for table and column names
- Use `INameCompatibility` interface for backward compatibility mapping
- Reference `BaseNameCompatibility.cs` for naming patterns

### Step 1 — Ask for TablePrefix
> **ALWAYS ask the user for the TablePrefix before generating.** Common prefixes: `NS_`

### Step 2 — Create Files in Order
1. **Domain Entity** (`Domains/MyEntity.cs`)
2. **BaseNameCompatibility** (`Data/BaseNameCompatibility.cs`)
3. **Entity Builder** (`Data/Builders/MyEntityBuilder.cs`)
4. **SchemaMigration** (`Data/SchemaMigration.cs`)

### Step 3 — Key Patterns/Templates

#### Domain Entity
```csharp
using Nop.Core;

namespace YourPlugin.Domains;

public class FAQItem : BaseEntity
{
    public string Question { get; set; }
    public string Answer { get; set; }
    public int DisplayOrder { get; set; }
    public bool Published { get; set; }
    public bool Deleted { get; set; }
    public DateTime CreatedOnUtc { get; set; }
    public DateTime? UpdatedOnUtc { get; set; }
}
```

#### BaseNameCompatibility — Table Name Mapping
```csharp
using Nop.Data.Mapping;

namespace YourPlugin.Data;

public class BaseNameCompatibility : INameCompatibility
{
    #region Properties

    public Dictionary<Type, string> TableNames => new()
    {
        { typeof(FAQItem), $"{PluginDefaults.TablePrefix}{nameof(FAQItem)}" }
    };

    public Dictionary<(Type, string), string> ColumnName => new()
    {
    };

    #endregion
}
```

#### Entity Builder

```csharp
using FluentMigrator.Builders.Create.Table;
using Nop.Data.Mapping.Builders;

namespace YourPlugin.Data.Builders;

public class FAQItemBuilder : NopEntityBuilder<FAQItem>
{
    public override void MapEntity(CreateTableExpressionBuilder table)
    {
        table
            .WithColumn(nameof(FAQItem.Question)).AsString(500).NotNullable()
            .WithColumn(nameof(FAQItem.Answer)).AsString(int.MaxValue).Nullable()
            .WithColumn(nameof(FAQItem.DisplayOrder)).AsInt32().NotNullable().WithDefaultValue(0)
            .WithColumn(nameof(FAQItem.Published)).AsBoolean().NotNullable().WithDefaultValue(true)
            .WithColumn(nameof(FAQItem.Deleted)).AsBoolean().NotNullable().WithDefaultValue(false)
            .WithColumn(nameof(FAQItem.CreatedOnUtc)).AsDateTime2().NotNullable()
            .WithColumn(nameof(FAQItem.UpdatedOnUtc)).AsDateTime2().Nullable();
    }
}
```
