# Widget Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Widgets"` |
| Namespace (nopCommerce) | `Nop.Plugin.Widgets.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Widgets.{Name}` |
| Core Interface | `IWidgetPlugin` |
| Class Naming | `{Name}Plugin` |

## Interface: `IWidgetPlugin`

Namespace: `Nop.Services.Cms`

### Methods

| Method | Signature |
|--------|-----------|
| `GetWidgetZonesAsync` | `Task<IList<string>> GetWidgetZonesAsync()` |
| `GetWidgetViewComponent` | `Type GetWidgetViewComponent(string widgetZone)` |

### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `HideInWidgetList` | `bool` | Hide from admin widget list when combined with another role (e.g., ITaxProvider) |

## Plugin Class Template

```csharp
public class {Name}Plugin : BasePlugin, IWidgetPlugin
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService, IWidgetService

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}Settings());

        if (!_widgetSettings.ActiveWidgetSystemNames.Contains({Name}Defaults.SystemName))
        {
            _widgetSettings.ActiveWidgetSystemNames.Add({Name}Defaults.SystemName);
            await _settingService.SaveSettingAsync(_widgetSettings);
        }

        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        if (_widgetSettings.ActiveWidgetSystemNames.Contains({Name}Defaults.SystemName))
        {
            _widgetSettings.ActiveWidgetSystemNames.Remove({Name}Defaults.SystemName);
            await _settingService.SaveSettingAsync(_widgetSettings);
        }

        await _settingService.DeleteSettingAsync<{Name}Settings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Widgets.{Name}");
        await base.UninstallAsync();
    }

    // --- IWidgetPlugin members ---

    public Task<IList<string>> GetWidgetZonesAsync()
    {
        return Task.FromResult<IList<string>>(new List<string>
        {
            // See Nop.Web.Framework.Infrastructure.PublicWidgetZones / AdminWidgetZones
            PublicWidgetZones.ProductDetailsBottom,
            PublicWidgetZones.HeaderLinksBefore,
            // Add your custom widget zone names here
        });
    }

    public Type GetWidgetViewComponent(string widgetZone)
    {
        // Return different ViewComponents for different widget zones if needed
        return typeof({Name}ViewComponent);
    }

    public bool HideInWidgetList => false;
}
```

## Unique Architecture

- **No public controllers**: Widgets never render via controller actions. All public rendering is done through ViewComponents.
- **ViewComponent-only rendering**: The plugin returns a ViewComponent type via `GetWidgetViewComponent()`. nopCommerce calls this ViewComponent at the specified widget zones.
- **Widget zone constants** come from:
  - `Nop.Web.Framework.Infrastructure.PublicWidgetZones` for storefront zones
  - `Nop.Web.Framework.Infrastructure.AdminWidgetZones` for admin zones
  - Or define custom zone names as constants
- **Active widget registration**: During `InstallAsync()`, add the plugin's SystemName to `WidgetSettings.ActiveWidgetSystemNames`. The ViewComponent is rendered because the system name is registered here.
- **Admin controller**: Named `Widgets{Name}Controller`, inherits `BaseAdminController`, has Configure GET/POST for settings
- **Single ViewComponent** can serve multiple widget zones by checking the `widgetZone` parameter in `InvokeAsync`
- **Required files**: `{Name}Plugin.cs`, `{Name}Defaults.cs`, `{Name}Settings.cs`, `Components/{Name}ViewComponent.cs`, `Areas/Admin/Controllers/Widgets{Name}Controller.cs`, `Views/Shared/Components/{Name}/Default.cshtml`

## Install/Uninstall Checklist

**Install:**
- Save default settings
- **Register in `WidgetSettings.ActiveWidgetSystemNames`** — critical step, missing this means the widget won't render
- Add localized resources

**Uninstall:**
- **Remove from `WidgetSettings.ActiveWidgetSystemNames`** — critical cleanup
- Delete settings
- Delete localized resources

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Components/{Name}ViewComponent.cs` | Yes (renders widget content) |
| `Areas/Admin/Controllers/` | Yes (settings page) |
| `Controllers/` (public) | No (widgets do not use public controllers) |
| `Domain/` | Only if custom data entities needed |
| `Data/` | Only if custom entities |
| `Factories/` | Only if complex admin model prep |
| `Models/` (public) | Rarely needed |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationViewComponent` instead of `NopViewComponent`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Never** create a public controller for widget rendering — always use ViewComponents
- **Must** register in `WidgetSettings.ActiveWidgetSystemNames` during install, or the widget won't render anywhere
- **Must** remove from `ActiveWidgetSystemNames` during uninstall to avoid orphaned entries
- `GetWidgetZonesAsync()` is called often — keep the list small and return a cached/static list
- The ViewComponent `Invoke` method signature must accept `(string widgetZone, object additionalData)` — this is the nopCommerce convention
- Settings are typically simple (enable toggle, API keys, pixel IDs)
