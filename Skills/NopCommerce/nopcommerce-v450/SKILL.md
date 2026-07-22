---
name: nopcommerce-v450
description: nopCommerce v4.50 plugin development changes since v4.40 — target framework, ORM switch to Linq2DB, .NET version, and notable plugin-related changes.
---

# nopCommerce v4.50 — Changes from v4.40

## When to Use

Activate this skill when:
- AGENTS.md specifies nopCommerce version 4.50
- Creating or modifying a plugin targeting nopCommerce v4.50
- Upgrading a plugin from v4.40 to v4.50

## Platform Changes from v4.40

### Target Framework
- Changed from **.NET 5** to **.NET 6 (LTS)**
- Requires Visual Studio 2022 (17.0.0 or above)
- Requires .NET 6 SDK (6.0.101) and ASP.NET Core Runtime (v6.0.1)

### ORM / Data Access — BREAKING CHANGE
- **Entity Framework (EF) replaced by Linq2DB**
- No navigation properties (Linq2DB does not support them) — use explicit joins via LINQ
- `IRepository<T>` pattern continues unchanged, but underlying implementation now uses Linq2DB
- `NopEntityTypeConfiguration<T>` mapping classes are no longer used
- Use `Nop.Data.Extensions` for `Create.TableFor<T>()` in FluentMigrator

### Database Connection
- `Microsoft.Data.SqlClient` changed default `Encrypt` from `false` to `true`
- Connection string in `appsettings.json` must include either:
  - `Encrypt=false`, or
  - `TrustServerCertificate=True`

### Plugin Lifecycle
- No changes to `BasePlugin`, `InstallAsync()`, `UninstallAsync()`, `UpdateAsync()` patterns
- FluentMigrator continues as the migration framework

### Breaking / Notable Changes

1. **Overnight ORM Change (EF → Linq2DB)**
   - Plugins using EF navigation properties must be rewritten
   - All database queries use Linq2DB now — expression trees may differ
   - Plugins with custom `IDbContext` or EF-based code will not compile

2. **New Integrations**
   - EasyPost shipping integration
   - what3words integration
   - Web API plugin (marketing feature — official REST API plugin)

3. **PayPal Commerce**
   - Added "Pay Later" messages feature

4. **General**
   - `appsettings.json` environment-specific file support
   - Anti-forgery token required on every public store page
   - Open redirect protection added

### Build and Deployment
- `.csproj` target framework must be updated to `net6.0`
- `SupportedVersions` in `plugin.json`: `["4.50"]`
- `Version` in `plugin.json` starts at `4.50.1`
