---
name: nopcommerce-v470
description: nopCommerce v4.70 plugin development changes since v4.60 — target framework, .NET version, and notable plugin-related changes.
---

# nopCommerce v4.70 — Changes from v4.60

## When to Use

Activate this skill when:
- AGENTS.md specifies nopCommerce version 4.70
- Creating or modifying a plugin targeting nopCommerce v4.70
- Upgrading a plugin from v4.60 to v4.70

## Platform Changes from v4.60

### Target Framework
- Changed from **.NET 7** to **.NET 8**
- Requires Visual Studio 2022 (17.9.0 or above)
- Requires .NET 8 SDK (8.0.204) and ASP.NET Core Runtime (v8.0.2)

### Plugin Lifecycle
- No changes to `BasePlugin`, `InstallAsync()`, `UninstallAsync()`, `UpdateAsync()` patterns

### Breaking / Notable Changes

1. **UPS Plugin — OAuth Migration**
   - UPS transitioned from access keys to OAuth (effective June 3, 2024)
   - UPS plugin users must reconfigure with OAuth settings
   - See https://developer.ups.com/oauth-developer-guide

2. **Bundling & Minimization Defaults**
   - Default values are now set for "Bundling & minimization"
   - If non-default values were previously used, reconfigure at: Admin > Configuration > Settings > App settings (or `appsettings.json`)

3. **Plugin Removals**
   - CyberSource plugin removed

4. **Renamed Plugins**
   - "Sendinblue" renamed to "Brevo" — requires reinstallation of the plugin

5. **New Integrations**
   - Omnisend integration added
   - OAuth2 authentication for email accounts

6. **Caching**
   - Redis-synced memory cache added with heartbeat-based locker

7. **General**
   - Architecture improvements and source code refactoring throughout
   - Deprecated `$(document).ready(handler)` usage progressively removed

### Build and Deployment
- `.csproj` target framework must be updated to `net8.0`
- `SupportedVersions` in `plugin.json`: `["4.70"]`
- `Version` in `plugin.json` starts at `4.70.1`
