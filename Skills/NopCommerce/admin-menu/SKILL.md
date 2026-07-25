---
name: admin-menu
description: Create admin menu items in nopCommerce plugins across all versions — covering both AdminMenuCreatedEvent (v4.80+) and IAdminMenuPlugin/ManageSiteMapAsync (v4.70 and below) patterns.
---

# Admin Menu Item Creation in nopCommerce

## When to Use

Use this skill when:
- Adding a new admin menu item from a plugin
- Deciding which pattern to use based on the target nopCommerce version
- Converting a plugin from the old `IAdminMenuPlugin` pattern to the new event-consumer pattern
- The version-specific skills (v4.40–v4.90) already loaded do not contain enough menu detail

---

## Version-Specific Rules — PICK ONE

### If target is nopCommerce **4.80 or above** (4.80, 4.90, 5.00+)

> **Mechanism: `AdminMenuCreatedEvent` + `IConsumer<T>` + `AdminMenuItem`**
> The menu is built in memory; plugins subscribe to `AdminMenuCreatedEvent` to inject items.

#### Step 1 — Create an Event Consumer class

```csharp
public class EventConsumer : IConsumer<AdminMenuCreatedEvent>
{
    private readonly IPermissionService _permissionService;

    public EventConsumer(IPermissionService permissionService)
    {
        _permissionService = permissionService;
    }

    public async Task HandleEventAsync(AdminMenuCreatedEvent eventMessage)
    {
        if (!await _permissionService.AuthorizeAsync(StandardPermission.Configuration.MANAGE_PLUGINS))
            return;

        eventMessage.RootMenuItem.InsertAfter("Local plugins",
            new AdminMenuItem
            {
                SystemName = "YourCustomSystemName",
                Title = "Plugin Title",
                Url = eventMessage.GetMenuItemUrl("ControllerName", "ActionName"),
                IconClass = "far fa-dot-circle",
                Visible = true,
            });
    }
}
```

> The consumer is **auto-discovered** by nopCommerce via DI scanning. No manual registration is needed.

#### URL Generation Helpers on `AdminMenuCreatedEvent`

| Method | Purpose |
|---|---|
| `GetMenuItemUrl(controller, action)` | Generates admin-area URL |
| `GetMenuItemUrl(controller, action, routeValues)` | URL with extra route values |

#### `AdminMenuItem` Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `SystemName` | `string` | Yes | Unique identifier for the menu item |
| `Title` | `string` | Yes | Display text (localize via `@T()`) |
| `Url` | `string` | Yes | Admin URL from `GetMenuItemUrl()` |
| `IconClass` | `string` | No | Font Awesome icon class |
| `Visible` | `bool` | No | Whether the item is rendered |
| `ChildNodes` | `IList<AdminMenuItem>` | No | Sub-menu items |

#### Positioning Methods on `RootMenuItem`

| Method | Description |
|---|---|
| `InsertBefore("ExistingSystemName", newItem)` | Insert before a known item |
| `InsertAfter("ExistingSystemName", newItem)` | Insert after a known item |
| `.ChildNodes.Add(newItem)` | Append as a child of the root |
| `parentItem.ChildNodes.InsertBefore(...)` / `InsertAfter(...)` | Position relative to siblings inside a parent |

#### Common Anchor System Names

| System Name | Location |
|---|---|
| `"Local plugins"` | Under "Local plugins" section in admin menu |
| `"Configuration"` | Configuration section |
| `"Catalog"` | Catalog section |
| `"Sales"` | Sales section |
| `"Customers"` | Customers section |
| `"Promotions"` | Promotions section |
| `"System"` | System section |
| `"Content"` | Content / CMS section |

#### Permission Enforcement

- Always gate menu visibility behind a permission check, never hardcode. Use **either**:
  - `StandardPermission.Configuration.MANAGE_PLUGINS` (generic)
  - A custom permission from your plugin's `IPermissionConfigManager`
- Example with custom permission:
  ```csharp
  if (!await _permissionService.AuthorizeAsync(MyPluginPermissionConfig.ACCESS_MY_FEATURE))
      return;
  ```
- `StandardPermission` lives in `Nop.Core.Domain.Security`

---

### If target is nopCommerce **4.70 or below** (4.40, 4.50, 4.60, 4.70)

> **Mechanism: `IAdminMenuPlugin` interface + `ManageSiteMapAsync(SiteMapNode)` + `SiteMapNode`**
> The menu is serialized from `_sitemap.config` (`~/Areas/Admin/sitemap.config`); plugins modify the in-memory node tree.

#### Step 1 — Implement `IAdminMenuPlugin` on the plugin main class

```csharp
public class CustomPlugin : BasePlugin, IAdminMenuPlugin
{
    public async Task ManageSiteMapAsync(SiteMapNode rootNode)
    {
        var menuItem = new SiteMapNode
        {
            SystemName = "YourCustomSystemName",
            Title = "Plugin Title",
            ControllerName = "YourController",
            ActionName = "List",
            IconClass = "far fa-dot-circle",
            Visible = true,
            RouteValues = new RouteValueDictionary { { "area", AreaNames.Admin } },
        };

        var pluginNode = rootNode.ChildNodes.FirstOrDefault(x => x.SystemName == "Third party plugins");
        if (pluginNode != null)
            pluginNode.ChildNodes.Add(menuItem);
        else
            rootNode.ChildNodes.Add(menuItem);
    }
}
```

#### `SiteMapNode` Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `SystemName` | `string` | Yes | Unique identifier |
| `Title` | `string` | Yes | Display text (localize via `@T()`) |
| `ControllerName` | `string` | Yes | Controller name (without "Controller" suffix) |
| `ActionName` | `string` | Yes | Action method name |
| `RouteValues` | `RouteValueDictionary` | Yes | Must include `{ "area", AreaNames.Admin }` for admin routes |
| `IconClass` | `string` | No | Font Awesome icon class |
| `Visible` | `bool` | No | Visibility flag |
| `ChildNodes` | `IList<SiteMapNode>` | No | Sub-menu items |
| `Url` | `string` | No | URL override (use instead of ControllerName/ActionName) |

> **IMPORTANT**: Always set `RouteValues = new RouteValueDictionary { { "area", AreaNames.Admin } }` — without this the URL will not resolve to the admin area.

#### Permission Enforcement (v4.70 and below)

- Use `IPermissionService.AuthorizeAsync()` inside `ManageSiteMapAsync` to gate visibility
- `ManageSiteMapAsync` is async — use `await`:
  ```csharp
  public async Task ManageSiteMapAsync(SiteMapNode rootNode)
  {
      if (!await _permissionService.AuthorizeAsync(StandardPermission.Configuration.MANAGE_PLUGINS))
          return;
      // ... add menu items
  }
  ```

---

## Cross-Cutting Rules (All Versions)

### Localization
- Always use `@T("Resource.Key")` for the `Title` property in Razor or pass a localized string
- Define resource keys in `GetPluginResources()` or via `ILocalizationService` during installation
- Example: `"Plugins.YourPlugin.MenuTitle"`

### Nested / Sub-Menus
Both patterns support parent-child nesting:

**v4.80+ (AdminMenuItem)**:
```csharp
var parent = new AdminMenuItem
{
    SystemName = "ParentPlugin",
    Title = "Parent",
    IconClass = "fas fa-cogs",
    Visible = true,
};
parent.ChildNodes.Add(new AdminMenuItem
{
    SystemName = "ChildItem",
    Title = "Child",
    Url = eventMessage.GetMenuItemUrl("Controller", "Action"),
    Visible = true,
});
eventMessage.RootMenuItem.ChildNodes.Add(parent);
```

**v4.70 and below (SiteMapNode)**:
```csharp
var parent = new SiteMapNode
{
    SystemName = "ParentPlugin",
    Title = "Parent",
    IconClass = "fas fa-cogs",
    Visible = true,
};
parent.ChildNodes.Add(new SiteMapNode
{
    SystemName = "ChildItem",
    Title = "Child",
    ControllerName = "Controller",
    ActionName = "Action",
    RouteValues = new RouteValueDictionary { { "area", AreaNames.Admin } },
    Visible = true,
});
rootNode.ChildNodes.Add(parent);
```

### Positioning Helpers (v4.70 and below)
There are no `InsertBefore`/`InsertAfter` helpers on `SiteMapNode`. Navigate `rootNode.ChildNodes` manually:
```csharp
var configNode = rootNode.ChildNodes.FirstOrDefault(x => x.SystemName == "Configuration");
if (configNode != null)
    configNode.ChildNodes.Add(menuItem);
```

### Common Anchor System Names (v4.70 and below)
| System Name | Location |
|---|---|
| `"Third party plugins"` | Dedicated section for plugin items |
| `"Configuration"` | Configuration section |
| `"Catalog"` | Catalog section |
| `"Sales"` | Sales section |
| `"Promotions"` | Promotions section |

---

## Summary — Pattern Selection Matrix

| nopCommerce Version | Interface/Pattern | Menu Item Class | URL Method | Registration |
|---|---|---|---|---|
| 4.80, 4.90, 5.00+ | `IConsumer<AdminMenuCreatedEvent>` | `AdminMenuItem` | `eventMessage.GetMenuItemUrl(...)` | Auto-DI-discovered |
| 4.40, 4.50, 4.60, 4.70 | `IAdminMenuPlugin` | `SiteMapNode` | `ControllerName` + `ActionName` + `RouteValues` | Implement on plugin class |

---

## What NOT to Do

- ❌ Do NOT modify `~/Areas/Admin/sitemap.config` directly — it is a core file
- ❌ Do NOT implement `IAdminMenuPlugin` on nopCommerce 4.80+ — the interface no longer exists
- ❌ Do NOT use `AdminMenuItem`/`AdminMenuCreatedEvent` on nopCommerce 4.70 and below — the class does not exist
- ❌ Do NOT hardcode URLs — always use `GetMenuItemUrl()` (v4.80+) or `ControllerName`/`ActionName` (v4.70-)
- ❌ Do NOT add menu items without a permission check
