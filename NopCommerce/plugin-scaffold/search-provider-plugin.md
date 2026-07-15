# Search Provider Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Search plugin manager"` |
| Namespace (nopCommerce) | `Nop.Plugin.Search.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Search.{Name}` |
| Core Interface | `IMiscPlugin` (marker only) |
| Class Naming | `{Name}SearchProvider` |

## Core Implementation Note

Despite the "Search" category name, nopCommerce does **not** have a dedicated `ISearchProvider` interface. Search provider plugins implement `IMiscPlugin` (marker interface with zero methods). The actual search functionality is implemented in the `Services/` layer and registered via `INopStartup`.

## Plugin Class Template

```csharp
public class {Name}SearchProvider : BasePlugin, IMiscPlugin
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService

    // The plugin class is intentionally thin — the heavy logic lives in services

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}SearchSettings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}SearchSettings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Search.{Name}");
        await base.UninstallAsync();
    }
}
```

## Unique Architecture

- **Thin plugin class**: The plugin class is intentionally minimal — less than 60 lines. It only handles lifecycle (install/uninstall) and configuration URL.
- **Search logic in Services layer**: The core search engine (e.g., Lucene, Elasticsearch, Algolia) is implemented as services:
  - `I{Name}SearchService` / `{Name}SearchService` — handles indexing and querying
  - `I{Name}SearchProvider` / `{Name}SearchProvider` — optional abstraction for search provider pattern
- **Registration via INopStartup**: The actual search provider integration is registered in the infrastructure layer, replacing or augmenting nopCommerce's default search behavior.
- **Admin controller**: Named `{Name}SearchController`, inherits `BaseAdminController`, provides settings management and optionally search index management (rebuild, status)
- **Public controllers**: Optional — only if the plugin has a custom search page or search widget
- **csharRequired files**: `{Name}SearchProvider.cs`, `{Name}Defaults.cs`, `{Name}SearchSettings.cs`, `Services/I{Name}SearchService.cs`, `Services/{Name}SearchService.cs`, `Infrastructure/NopStartup.cs`, `Areas/Admin/Controllers/{Name}SearchController.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings (index path, API keys, search configuration)
- Add localized resources
- Optionally register scheduled tasks for index rebuilding

**Uninstall:**
- Delete settings
- Delete localized resources
- Remove scheduled tasks
- Optionally clean up index files on disk

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Areas/Admin/Controllers/` | Yes (settings + index management) |
| `Controllers/` (public) | Only if custom public search UI |
| `Components/` | Only if widget zone search widget |
| `Services/` | Yes (search engine logic) |
| `Infrastructure/NopStartup.cs` | Yes (register services and potentially replace default search) |
| `Domain/` | Only if custom search-related entities |
| `Data/` | Only if custom entities |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **The plugin class itself should be thin** — don't put search logic in the plugin class
- Search providers typically need to hook into nopCommerce's search pipeline — this is done via service registration in `INopStartup`, not by modifying core
- If your search provider replaces the default search, register it with a higher `Order` value in `INopStartup` so it overrides the default implementation
- Index management (rebuild, incremental update) is often handled via scheduled tasks
- Consider thread safety — search indexing may run as a background task while customers browse
