---
name: nopcommerce-v460
description: nopCommerce v4.60 plugin development changes since v4.50 — target framework, .NET version, and notable plugin-related changes.
---

# nopCommerce v4.60 — Changes from v4.50

## When to Use

Activate this skill when:
- AGENTS.md specifies nopCommerce version 4.60
- Creating or modifying a plugin targeting nopCommerce v4.60
- Upgrading a plugin from v4.50 to v4.60

## Platform Changes from v4.50

### Target Framework
- Changed from **.NET 6 (LTS)** to **.NET 7**
- Requires Visual Studio 2022 (17.4.0 or above)
- Requires .NET 7 SDK (7.0.101) and ASP.NET Core Runtime (v7.0.1)

### ORM / Data Access
- No change — still uses **Linq2DB** (not Entity Framework)

### Plugin Lifecycle
- No changes to `BasePlugin`, `InstallAsync()`, `UninstallAsync()`, `UpdateAsync()` patterns

### Breaking / Notable Changes

1. **Plugin Removals**
   - **PayPal Standard** plugin removed
   - **ShipStation** plugin removed
   - **EasyPost** shipping plugin removed

2. **New Security Features**
   - Multi-factor authentication (MFA) support added
   - Customers required to re-login on all devices after password change

3. **New Features Impacting Plugin Logic**
   - Product video support added (new field on product entities)
   - `IsActive` property added to discounts (ability to enable/disable discounts)
   - VAT number entry allowed in guest checkout
   - reCAPTCHA added to guest checkout
   - Robots.txt editable from admin area
   - New activity log types added (export/import of categories, customers, manufacturers, etc.)
   - Localized export/import for Products, Manufacturers, Categories (multi-language data)

4. **Frontend**
   - Display all pictures on catalog pages
   - Instagram added to default social media links
   - Search products by manufacturer names and category names

### Build and Deployment
- `.csproj` target framework must be updated to `net7.0`
- `SupportedVersions` in `plugin.json`: `["4.60"]`
- `Version` in `plugin.json` starts at `4.60.1`
