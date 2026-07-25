---
name: permissions
description: Define and enforce custom permissions in a nopCommerce plugin — covering IPermissionConfigManager (v4.80+), IPermissionProvider (v4.70-), [CheckPermission] attribute, and IPermissionService.
---

# Permissions in nopCommerce Plugins

## When to Use

Use this skill when:
- Adding custom permissions to a plugin
- Securing controller actions with `[CheckPermission]`
- Gating admin menu items behind permission checks
- Performing programmatic authorization in services or event consumers
- Installing/uninstalling permissions during plugin lifecycle

---

## Version-Specific Rules — PICK ONE

### If target is nopCommerce **4.80 or above** (4.80, 4.90, 5.00+)

> **API: `IPermissionConfigManager` interface**
> Permissions are auto-registered into the ACL section on application start — no explicit `InstallAsync()` call needed.

#### Step 1 — Define Permission Constants + Config Manager

```csharp
public partial class MyPluginPermissionConfigManager : IPermissionConfigManager
{
    public const string ACCESS_MY_PLUGIN = "AccessMyPlugin";

    public IList<PermissionConfig> AllConfigs =>
        new List<PermissionConfig>
        {
            new("Access My Plugin", ACCESS_MY_PLUGIN, nameof(StandardPermission.System), NopCustomerDefaults.AdministratorsRoleName)
        };
}
```

| Parameter | Description |
|---|---|
| `name` | Display name shown in ACL admin page |
| `systemName` | Unique string constant used in `[CheckPermission]` and `AuthorizeAsync()` |
| `category` | Grouping for the ACL page (use `nameof(StandardPermission.System)` or a custom category string) |
| `defaultCustomerRoles` | Role(s) granted by default (use `NopCustomerDefaults.AdministratorsRoleName`) |

> Permissions are **auto-installed** — no code needed in `InstallAsync()`.

#### Step 2 — Secure Controller Actions

```csharp
[CheckPermission(MyPluginPermissionConfigManager.ACCESS_MY_PLUGIN)]
public virtual async Task<IActionResult> Configure()
{
    return View();
}
```

Or programmatically:

```csharp
if (!await _permissionService.AuthorizeAsync(MyPluginPermissionConfigManager.ACCESS_MY_PLUGIN))
    return AccessDeniedView();
```

#### Step 3 — Clean Up on Uninstall

```csharp
public override async Task UninstallAsync()
{
    var permissionRecord = (await _permissionService.GetAllPermissionRecordsAsync())
        .FirstOrDefault(x => x.SystemName == MyPluginPermissionConfigManager.ACCESS_MY_PLUGIN);
    if (permissionRecord != null)
        await _permissionService.DeletePermissionRecordAsync(permissionRecord);

    await base.UninstallAsync();
}
```

---

### If target is nopCommerce **4.70 or below** (4.40, 4.50, 4.60, 4.70)

> **API: `IPermissionProvider` interface**
> Permissions must be explicitly installed in `InstallAsync()` and uninstalled in `UninstallAsync()`.

#### Step 1 — Define Permission Provider

```csharp
public partial class MyPluginPermissionProvider : IPermissionProvider
{
    public static readonly PermissionRecord AccessMyPlugin = new()
    {
        Name = "Access My Plugin",
        SystemName = "AccessMyPlugin",
        Category = "Standard"
    };

    public virtual IEnumerable<PermissionRecord> GetPermissions()
    {
        return new[] { AccessMyPlugin };
    }

    public virtual HashSet<(string systemRoleName, PermissionRecord[] permissions)> GetDefaultPermissions()
    {
        return new()
        {
            (NopCustomerDefaults.AdministratorsRoleName, new[] { AccessMyPlugin })
        };
    }
}
```

#### Step 2 — Install in `InstallAsync()`

```csharp
public override async Task InstallAsync()
{
    await _permissionService.InstallPermissionsAsync(new MyPluginPermissionProvider());
    await base.InstallAsync();
}
```

#### Step 3 — Secure Controller Actions

```csharp
if (!await _permissionService.AuthorizeAsync(MyPluginPermissionProvider.AccessMyPlugin.SystemName))
    return AccessDeniedView();
```

#### Step 4 — Clean Up on Uninstall

```csharp
public override async Task UninstallAsync()
{
    var permissionRecord = (await _permissionService.GetAllPermissionRecordsAsync())
        .FirstOrDefault(x => x.SystemName == MyPluginPermissionProvider.AccessMyPlugin.SystemName);
    if (permissionRecord != null)
        await _permissionService.DeletePermissionRecordAsync(permissionRecord);

    await base.UninstallAsync();
}
```

---

## Cross-Cutting Rules (All Versions)

### Available Standard Permissions

Out-of-the-box permissions are in `StandardPermission` class (`Nop.Services.Security` namespace):

```csharp
[CheckPermission(StandardPermission.Configuration.MANAGE_SETTINGS)]
```

### Permission Checks in Menu Items (v4.80+)

Always gate admin menu items behind permission checks:

```csharp
public async Task HandleEventAsync(AdminMenuCreatedEvent eventMessage)
{
    if (!await _permissionService.AuthorizeAsync(MyPluginPermissionConfigManager.ACCESS_MY_PLUGIN))
        return;

    eventMessage.RootMenuItem.InsertAfter("Local plugins",
        new AdminMenuItem
        {
            SystemName = "MyPlugin",
            Title = "My Plugin",
            Url = eventMessage.GetMenuItemUrl("MyPlugin", "Configure"),
            IconClass = "far fa-dot-circle",
            Visible = true,
        });
}
```

### UI Visibility via Model Factory

Do NOT inject `IPermissionService` into Razor views. Check permissions in the model factory instead:

```csharp
public async Task<ConfigureModel> PrepareConfigureModelAsync()
{
    return new ConfigureModel
    {
        CanManageSensitiveSettings = await _permissionService.AuthorizeAsync(
            MyPluginPermissionConfigManager.ACCESS_MY_PLUGIN)
    };
}
```

```html
@if (Model.CanManageSensitiveSettings)
{
    <nop-card asp-name="sensitive-settings" ...>
    </nop-card>
}
```

---

## Summary — Pattern Selection Matrix

| Version | Interface | Auto-Installed? | Install Code | Cleanup |
|---|---|---|---|---|
| 4.80, 4.90, 5.00+ | `IPermissionConfigManager` | Yes | None needed | Delete on uninstall |
| 4.40, 4.50, 4.60, 4.70 | `IPermissionProvider` | No | `InstallPermissionsAsync()` in `InstallAsync()` | Delete on uninstall |

---

## What NOT to Do

- ❌ Do NOT implement `IPermissionProvider` on nopCommerce 4.80+ — it was removed
- ❌ Do NOT call `InstallPermissionsAsync()` on v4.80+ — permissions are auto-registered via `IPermissionConfigManager`
- ❌ Do NOT inject `IPermissionService` directly into Razor views
- ❌ Do NOT hardcode permission strings in `[CheckPermission]` — always reference constants
- ❌ Do NOT skip permission cleanup in `UninstallAsync()` — stale records pollute the ACL page
