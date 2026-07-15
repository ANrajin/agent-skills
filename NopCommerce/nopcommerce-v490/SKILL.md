---
name: nopcommerce-v490
description: nopCommerce v4.90 plugin development conventions — architecture, permissions, migrations, scheduled tasks, localization, and error handling.
---

# nopCommerce v4.90 Rules

## When to Use

Activate this skill when:
- AGENTS.md specifies nopCommerce version 4.90
- Creating or modifying a plugin targeting nopCommerce v4.90
- Writing version-specific code (migrations, DI, controllers) for v4.90

## Platform Overview
- Target framework: .NET 9.0
- ORM: Linq2DB (NOT Entity Framework)
- Migrations: FluentMigrator
- Validation: FluentValidation
- Database support: MS SQL Server, MySQL, PostgreSQL

## Plugin Architecture

### Plugin Structure Requirements
- Every plugin requires `.csproj` targeting .NET 9.0 and `plugin.json` metadata file
- Plugin class must inherit from `BasePlugin`
- Override `InstallAsync()`, `UninstallAsync()`, `UpdateAsync()` for lifecycle hooks
- Always call `base.InstallAsync()` / `base.UninstallAsync()` when overriding

### plugin.json Configuration
```json
{
  "Group": "Payment methods",
  "FriendlyName": "My Custom Plugin",
  "SystemName": "Payments.MyPlugin",
  "Version": "4.90.1",
  "SupportedVersions": ["4.90"],
  "Author": "Your Company",
  "DisplayOrder": 1,
  "FileName": "Nop.Plugin.Payments.MyPlugin.dll",
  "Description": "Plugin description",
  "LimitedToStores": [],
  "LimitedToCustomerRoles": [],
  "DependsOnSystemNames": []
}
```

### Plugin Versioning
- Always update `Version` in plugin.json when adding migrations or localized resources
- Use `UpdateAsync(string currentVersion, string targetVersion)` for version-specific upgrade logic

## Scheduled Tasks

### Task Implementation
- Implement `IScheduleTask` interface
- Register task in database during plugin installation using `IScheduleTaskService`
- Type format: `"Namespace.TaskClassName, AssemblyName"`

```csharp
public class MyScheduledTask : IScheduleTask
{
    public async Task ExecuteAsync()
    {
    }
}
```

### Task Registration
```csharp
await _scheduleTaskService.InsertTaskAsync(new ScheduleTask
{
    Name = "My Task",
    Seconds = 3600,
    Type = "Nop.Plugin.Misc.MyPlugin.Services.MyScheduledTask, Nop.Plugin.Misc.MyPlugin",
    Enabled = true,
    StopOnError = false
});
```

## Settings Management

### Plugin Settings
- Create settings class inheriting from `ISettings`
- Use `ISettingService` to load/save settings
- Settings are scoped per store by default

```csharp
public class MyPluginSettings : ISettings
{
    public string ApiKey { get; set; }
    public bool UseSandbox { get; set; }
}

var settings = await _settingService.LoadSettingAsync<MyPluginSettings>();

await _settingService.SaveSettingAsync(settings);
```

## Error Handling

### Logging
- Use `ILogger` from `Nop.Services.Logging` (NOT `Microsoft.Extensions.Logging.ILogger<T>`)
- Wrap external system calls in try/catch blocks
- Always localize log messages via `ILocalizationService.GetResourceAsync()` — never hardcode them
- Log errors with `await _logger.ErrorAsync(await _localizationService.GetResourceAsync("..."), ex)`

```csharp
try
{
    await _externalService.CallAsync();
}
catch (Exception ex)
{
    await _logger.ErrorAsync(
        await _localizationService.GetResourceAsync("Plugins.MyPlugin.Error.ExternalService"), ex);
    throw;
}
```

## DateTime Handling

### Rules
- Store all dates in UTC in the database
- Use `DateTime.UtcNow` for new records
- Property names should end with `Utc` suffix
- Use `_dateTimeHelper.ConvertToUserTimeAsync()` for display

## Build and Deployment

### Project File (.csproj)
- Target `net9.0`
- Reference `Nop.Web.Framework` project
- Set output path to plugin directory

### Assembly References
- Do NOT copy nopCommerce assemblies to plugin output
- Use `<Private>false</Private>` for nopCommerce references
