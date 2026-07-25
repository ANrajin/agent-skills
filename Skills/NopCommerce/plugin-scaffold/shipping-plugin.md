# Shipping Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Shipping rate computation methods"` |
| Namespace (nopCommerce) | `Nop.Plugin.Shipping.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Shipping.{Name}` |
| Core Interface | `IShippingRateComputationMethod` |
| Class Naming | `{Name}ComputationMethod` |

## Interface: `IShippingRateComputationMethod`

Namespace: `Nop.Services.Shipping`

### Methods

| Method | Signature |
|--------|-----------|
| `GetShippingOptionsAsync` | `Task<GetShippingOptionRequest> GetShippingOptionsAsync(GetShippingOptionRequest getShippingOptionRequest)` |
| `GetFixedRateAsync` | `Task<decimal?> GetFixedRateAsync(GetShippingOptionRequest getShippingOptionRequest)` |
| `GetShipmentTrackerAsync` | `Task<IShipmentTracker> GetShipmentTrackerAsync()` |

### Parameter/Result Types

`GetShippingOptionRequest` contains:
- `ShippingAddress` / `BillingAddress`
- `Customer`
- `Items` — `IList<ShoppingCartItem>`
- `CountryFrom` / `StateProvinceFrom` / `ZipPostalCodeFrom`
- `CountryTo` / `StateProvinceTo` / `ZipPostalCodeTo`
- `WarehouseId`
- `StoreId`

`GetShippingOptionRequest` result returns:
- `ShippingOptions` — `IList<ShippingOption>` (Name, Rate, Description, TransitDays, DisplayOrder)
- `Errors` — list of errors
- `ShippingOptionType` — specifies if rate is generic or type-specific

`IShipmentTracker` provides real-time tracking:
- `GetTrackingEventsAsync(string trackingNumber)`
- `GetTrackingUrl(string trackingNumber)`
- `IsSupported` — whether tracking is available

## Plugin Class Template

```csharp
public class {Name}ComputationMethod : BasePlugin, IShippingRateComputationMethod
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService, ILogger

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}ShippingSettings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}ShippingSettings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Shipping.{Name}");
        await base.UninstallAsync();
    }

    // --- IShippingRateComputationMethod members ---

    public async Task<GetShippingOptionRequest> GetShippingOptionsAsync(GetShippingOptionRequest request)
    {
        var result = new GetShippingOptionRequest();

        // Calculate shipping options based on destination, weight, dimensions
        // Call external API or compute locally

        var shippingOption = new ShippingOption
        {
            Name = await _localizationService.GetResourceAsync(...),
            Rate = calculatedRate,
            Description = "Estimated delivery: X business days",
            TransitDays = 5
        };

        result.ShippingOptions.Add(shippingOption);
        return result;
    }

    // Return null if not a fixed-rate-only provider
    public Task<decimal?> GetFixedRateAsync(GetShippingOptionRequest request)
    {
        return Task.FromResult<decimal?>(null);
    }

    // Return null if no real-time tracking available
    public Task<IShipmentTracker> GetShipmentTrackerAsync()
    {
        return Task.FromResult<IShipmentTracker>(null);
    }
}
```

## Unique Architecture

- **3 methods, 0 properties** — lean interface focused on rate calculation
- **Fixed Rate vs Computation**: Use `GetFixedRateAsync` for simple flat-rate plugins (return the rate). Return `null` for calculated/API-based shipping.
- **Shipment Tracking**: Optional via `IShipmentTracker`. Only implement if your carrier provides real-time tracking APIs.
- **Admin controller**: Named `Shipping{Name}Controller`, inherits `BaseAdminController`
- **No public controllers**: Shipping rate computation is server-side, no public UI needed
- **Domain/Data**: Optional — needed for storing rate overrides, weight tables, etc. (e.g., FixedByWeightByTotal stores shipping-by-weight records)
- **Required files**: `{Name}ComputationMethod.cs`, `{Name}Defaults.cs`, `{Name}ShippingSettings.cs`, `Areas/Admin/Controllers/Shipping{Name}Controller.cs`, `Services/{Name}Service.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings (API keys, package dimensions, handling fees)
- Add localized resources
- Optionally install schema migration if storing rate records

**Uninstall:**
- Delete settings
- Delete localized resources
- Optionally remove schema migration tables (be careful with rate records that may be referenced)

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Areas/Admin/Controllers/` | Yes (settings page) |
| `Controllers/` (public) | No |
| `Components/` | No (no public rendering) |
| `Services/` | Yes (rate calculation / API client) |
| `Domain/` | Only if custom rate entities |
| `Data/` | Only if custom entities |
| `Factories/` | Only if complex admin view models |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Never** modify the `GetShippingOptionRequest` parameter — it's read-only
- `GetShippingOptionsAsync` is called per-item in the cart during checkout — consider caching rate lookups per-request
- Return `null` from `GetFixedRateAsync` unless your plugin charges a single flat rate regardless of destination/weight
- `IShipmentTracker` is optional — most shipping plugins don't need it
- Shipping rates are `decimal` — store with standard precision
- For warehouse-specific rates, check `request.WarehouseId`
