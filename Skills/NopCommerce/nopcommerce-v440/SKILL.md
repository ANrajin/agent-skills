---
name: nopcommerce-v440
description: nopCommerce v4.40 plugin development changes since v4.30 — target framework, async methods, FluentMigrator, and notable plugin-related changes.
---

# nopCommerce v4.40 — Changes from v4.30

## When to Use

Activate this skill when:
- AGENTS.md specifies nopCommerce version 4.40
- Creating or modifying a plugin targeting nopCommerce v4.40
- Upgrading a plugin from v4.30 to v4.40

## Platform Changes from v4.30

### Target Framework
- Changed from **.NET Core 3.1** to **.NET 5**
- Requires Visual Studio 2019 (16.8+) or above (or VS 2022)
- Requires .NET 5 SDK and ASP.NET Core Runtime v5.0

### Async Methods — BREAKING CHANGE
- **All methods in nopCommerce became async** — this is the single biggest change
- Controllers, services, and repository calls now use `async Task` / `Task<T>` signatures
- Plugin code must use `await` for all nopCommerce service calls
- `IActionResult` replaced by `async Task<IActionResult>` in all controller actions
- Synchronous wrappers removed — plugins must be fully async

### Migrations — BREAKING CHANGE
- **No more SQL upgrade scripts** — all migrations handled automatically via FluentMigrator on first application start
- Plugins must use `FluentMigrator` with `[NopSchemaMigration]` or `[NopMigration]` attributes
- Migration attribute format: `[NopSchemaMigration("yyyy/MM/dd HH:mm:ss:fffffff", "description", MigrationProcessType.Installation)]`
- `Create.TableFor<TEntity>()` extension from `Nop.Data.Extensions`

### Performance
- ~30% performance increase from async methods and .NET 5 optimizations
- Full web farm support added
- Redis caching performance improvements

### Plugin Lifecycle
- No changes to `BasePlugin`, `InstallAsync()`, `UninstallAsync()`, `UpdateAsync()` patterns
- All override methods are already async — ensure `await base.InstallAsync()` etc.

### Breaking / Notable Changes

1. **Async Everywhere**
   - Synchronous methods removed from services
   - Controller actions must return `async Task<IActionResult>`
   - `_logger.ErrorAsync()`, `_settingService.LoadSettingAsync()`, etc. are the standard

2. **FluentMigrator Standardized**
   - Plugin migrations use `ForwardOnlyMigration` or `Migration` base classes
   - `NopSchemaMigration` attribute marks migrations for installation
   - `NopMigration` attribute marks migrations for upgrade (version-specific)
   - Migration numbering uses timestamp-based unique identifiers

3. **New Features**
   - PayPal Commerce plugin introduced (later renamed/updated in subsequent versions)
   - Various performance and caching improvements

### Build and Deployment
- `.csproj` target framework must be updated to `net5.0`
- `SupportedVersions` in `plugin.json`: `["4.40"]`
- `Version` in `plugin.json` starts at `4.40.1`
