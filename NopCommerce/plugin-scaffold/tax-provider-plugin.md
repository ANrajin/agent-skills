# Tax Provider Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Tax providers"` |
| Namespace (nopCommerce) | `Nop.Plugin.Tax.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Tax.{Name}` |
| Core Interface | `ITaxProvider` |
| Class Naming | `{Name}TaxProvider` |

## Interface: `ITaxProvider`

Namespace: `Nop.Services.Tax`

### Methods

| Method | Signature |
|--------|-----------|
| `GetTaxRateAsync` | `Task<TaxRateResult> GetTaxRateAsync(TaxRateRequest taxRateRequest)` |
| `GetTaxTotalAsync` | `Task<TaxTotalResult> GetTaxTotalAsync(TaxTotalRequest taxTotalRequest)` |

### Parameter/Result Types

`TaxRateRequest` contains:
- `Address` — shipping address (may be null)
- `Customer` — the customer
- `Product` — the product (may be null)
- `Quantity` — quantity
- `TaxCategoryId` — the tax category
- `CurrentStoreId` / `StoreId`

`TaxRateResult` contains:
- `TaxRate` — decimal rate
- `Errors` — list of string errors

`TaxTotalRequest` contains:
- `ShoppingCart` — `IList<ShoppingCartItem>`
- `Customer` — the customer
- `StoreId`
- `UseCache` — whether caching is allowed

`TaxTotalResult` contains:
- `TaxTotal` — decimal total
- `TaxRates` — `SortedDictionary<decimal, decimal>` (rate → amount)
- `Errors` — list of string errors

## Plugin Class Template

```csharp
public class {Name}TaxProvider : BasePlugin, ITaxProvider
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService, ILogger

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}TaxSettings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}TaxSettings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Tax.{Name}");
        await base.UninstallAsync();
    }

    // --- ITaxProvider members ---

    public async Task<TaxRateResult> GetTaxRateAsync(TaxRateRequest request)
    {
        if (request.Address == null)
            return new TaxRateResult { TaxRate = 0 };

        // Calculate based on address, product tax category, etc.
        var rate = await CalculateRateAsync(request);
        return new TaxRateResult { TaxRate = rate };
    }

    public async Task<TaxTotalResult> GetTaxTotalAsync(TaxTotalRequest request)
    {
        // For cart-level tax calculation
        // Optionally cache the result per request in HttpContext.Items to avoid duplicate calls
        var result = await CalculateCartTaxAsync(request);
        return result;
    }
}
```

## Unique Architecture

- **Simple interface** — only 2 methods, no properties
- **Optional dual-interface pattern**: Some tax providers also implement `IWidgetPlugin` to inject admin UI components (e.g., entity use code selectors, exemption certificates). Set `HideInWidgetList => true` to avoid appearing as a widget.
- **Scheduled tasks**: Complex tax providers (e.g., Avalara) register scheduled tasks for tax rate table downloads or batch processing
- **Caching**: `GetTaxRateAsync` and `GetTaxTotalAsync` are called frequently during checkout. Cache results per-request using `HttpContext.Items` to avoid redundant API calls.
- **Admin controller**: Named `Tax{Name}Controller`, inherits `BaseAdminController`
- **No public controllers**: Tax providers don't have public-facing pages
- **Required files**: `{Name}TaxProvider.cs`, `{Name}Defaults.cs`, `{Name}TaxSettings.cs`, `Areas/Admin/Controllers/Tax{Name}Controller.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings (API keys, connection urls, tax codes mapping)
- Add localized resources
- Optionally register scheduled tasks for tax rate syncing
- Optionally register widget zone rendering (if dual-interface pattern)

**Uninstall:**
- Delete settings
- Delete localized resources
- Remove any scheduled tasks
- Unregister from widget zones (if dual-interface)

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Areas/Admin/Controllers/` | Yes (settings page) |
| `Controllers/` (public) | No |
| `Components/` | Only if dual-interface with `IWidgetPlugin` |
| `Domain/` | Only if custom entities (tax rate overrides, exemption records) |
| `Data/` | Only if custom entities |
| `Services/` | Yes (API client, rate calculation logic) |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Do not** assume `request.Address` is non-null in `GetTaxRateAsync` — it can be null for product-level calculations
- `GetTaxTotalAsync` is called per-page-load during checkout — design for performance and cache aggressively
- Tax rates are `decimal` — use a precision of at least 4 decimal places
- **Do not** store calculated tax in order records from the tax provider — nopCommerce handles this
- For dual-interface (Tax + Widget), remember to set `HideInWidgetList => true`
