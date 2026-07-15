# Pickup Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Pickup points"` |
| Namespace (nopCommerce) | `Nop.Plugin.Pickup.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Pickup.{Name}` |
| Core Interface | `IPickupPointProvider` |
| Class Naming | `{Name}PickupPointProvider` |

## Interface: `IPickupPointProvider`

Namespace: `Nop.Services.Shipping.Pickup`

### Methods

| Method | Signature |
|--------|-----------|
| `GetPickupPointsAsync` | `Task<GetPickupPointsResult> GetPickupPointsAsync(GetPickupPointsRequest getPickupPointsRequest)` |
| `GetShipmentTrackerAsync` | `Task<IShipmentTracker> GetShipmentTrackerAsync()` |

### Parameter/Result Types

`GetPickupPointsRequest` contains:
- `SearchTerm` — optional search/filter
- `Address` or `ShippingAddress`
- `Customer`
- `StoreId`
- `PageIndex` / `PageSize`

`GetPickupPointsResult` contains:
- `PickupPoints` — `IList<PickupPoint>` (Name, Description, Address, OpeningHours, Fee, TransitDays, PickupFee, ProviderSystemName)
- `Errors` — list of errors
- `TotalCount`

## Plugin Class Template

```csharp
public class {Name}PickupPointProvider : BasePlugin, IPickupPointProvider
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService, IRepository<{Name}Pickup> (if custom entity)

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}PickupSettings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}PickupSettings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Pickup.{Name}");
        await base.UninstallAsync();
    }

    // --- IPickupPointProvider members ---

    public async Task<GetPickupPointsResult> GetPickupPointsAsync(GetPickupPointsRequest request)
    {
        // Query your pickup locations (from DB or external API)
        var pickupPoints = await _pickupRepository.GetAllAsync(query =>
        {
            // Filter by request.SearchTerm, request.StoreId, etc.
        });

        var result = new GetPickupPointsResult
        {
            PickupPoints = pickupPoints.Select(p => new PickupPoint
            {
                Name = p.Name,
                Description = p.Description,
                Address = p.FullAddress,
                OpeningHours = p.OpeningHours,
                Fee = p.PickupFee,
                ProviderSystemName = {Name}Defaults.SystemName
            }).ToList()
        };

        return result;
    }

    // Return null if no real-time tracking available
    public Task<IShipmentTracker> GetShipmentTrackerAsync()
    {
        return Task.FromResult<IShipmentTracker>(null);
    }
}
```

## Unique Architecture

- **Similar to shipping but for pickup**: Instead of calculating rates, you return a list of physical pickup locations
- **Pickup points can be stored in DB** (via custom entity + migration) or fetched from an external API (e.g., carrier location finder)
- **Admin CRUD** is common: a management UI for store owners to add/edit/delete pickup locations
- **Admin controller**: Named `Pickup{Name}Controller` or `{Name}PickupPointController`, inherits `BaseAdminController`
- **No public controllers**: Pickup points are fetched server-side during checkout
- **Domain/Data**: Typically needed to store pickup location data
- **Required files**: `{Name}PickupPointProvider.cs`, `{Name}Defaults.cs`, `{Name}PickupSettings.cs`, `Areas/Admin/Controllers/Pickup{Name}Controller.cs`, `Domain/{Name}PickupPoint.cs`, `Data/SchemaMigration.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings
- Add localized resources
- Install schema for pickup points table

**Uninstall:**
- Delete settings
- Delete localized resources
- Drop pickup points table (or keep — depending on user preference)

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Areas/Admin/Controllers/` | Yes (settings + pickup point management) |
| `Controllers/` (public) | No |
| `Components/` | No |
| `Domain/` | Yes (pickup point entity) |
| `Data/` | Yes (schema migration, entity builder) |
| `Services/` | Yes (CRUD service for pickup points) |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- Pickup points return a **fee** (not a rate) — this is the additional charge (or discount) for choosing pickup over shipping
- The `PickupPoint` class includes `Fee` which can be `decimal` — set to `0` for free pickup
- `GetShipmentTrackerAsync()` is rarely used for pickup plugins — return null unless your pickup provider also handles shipment tracking
- Pickup points don't need tracking numbers in the traditional sense — but if you store order-pickup associations, create a separate entity
