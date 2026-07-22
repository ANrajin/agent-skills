# Misc Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Misc"` |
| Namespace (nopCommerce) | `Nop.Plugin.Misc.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Misc.{Name}` |
| Core Interface | `IMiscPlugin` |
| Class Naming | `{Name}Plugin` |

## Interface: `IMiscPlugin`

Namespace: `Nop.Services.Plugins`

`IMiscPlugin` is a **marker interface** — it has zero methods. It only marks the plugin as a miscellaneous plugin for categorization.

The plugin class provides all functionality through injected services and optionally implements additional interfaces.

## Plugin Class Template

```csharp
public class {Name}Plugin : BasePlugin, IMiscPlugin
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}Settings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}Settings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Misc.{Name}");
        await base.UninstallAsync();
    }
}
```

## Unique Architecture

- **Most flexible plugin type** — IMiscPlugin has no required methods, making it suitable for anything that doesn't fit other categories
- **Can combine with other interfaces**: Common combinations include `IMiscPlugin + IWidgetPlugin` (e.g., Brevo for email marketing + tracking pixel), or `IMiscPlugin + IPaymentMethod` (rare)
- **Varies wildly in scope**:
  - Backend-only (e.g., AzureBlob — no public UI beyond config)
  - Full CRUD with admin data management (e.g., Zettle)
  - Hybrid with widgets (e.g., OmnibusDirective, Brevo)
- **Admin controller**: Typically named `{Name}AdminController` or `{Name}Controller`, inherits `BaseAdminController`
- **Public controllers**: Optional — only if the plugin has public-facing features
- **Domain/Data**: Optional — only if the plugin stores its own entities
- **Scheduled tasks**: Optional — common for sync/data import tasks
- **Message templates**: Optional — if the plugin sends custom email notifications

## Install/Uninstall Checklist

**Install (pick what applies):**
- Save default settings
- Add localized resources
- Register scheduled tasks
- Create message templates via `_messageTemplateService.InsertMessageTemplateAsync()`
- Register in widget zones (if combined with `IWidgetPlugin`)

**Uninstall (pick what applies):**
- Delete settings
- Delete localized resources
- Remove scheduled tasks
- Delete message templates
- Unregister from widget zones

## Files to Include vs Exclude

| File/Directory | Required? |
|---------------|-----------|
| `Areas/Admin/Controllers/` | Yes (at minimum settings page) |
| `Controllers/` (public) | Only if public features |
| `Components/` | Only if combined with `IWidgetPlugin` |
| `Domain/` | Only if custom entities |
| `Data/` | Only if custom entities |
| `Services/` | Usually yes (business logic) |
| `Factories/` | Only if complex model prep |
| `Events/` | Only if reacting to nopCommerce events |
| `Views/` (public) | Only if public features |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `NopStationPublicController` for public controllers
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Define permissions via `IPermissionConfigManager`
- Admin menu via `AdminMenuCreatedEventConsumer` with `NopStationAdminMenuItem`
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Not a catch-all** — if the plugin fits another category (Payment, Tax, Shipping, Widget), use that instead
- The marker interface gives maximum freedom, but also zero guidance — you must define your own architecture
- If the plugin needs public rendering, consider combining `IWidgetPlugin` with `IMiscPlugin` rather than creating public controller actions for embedded content
- Settings class is optional — some misc plugins have no configuration (e.g., pure service integrations)
