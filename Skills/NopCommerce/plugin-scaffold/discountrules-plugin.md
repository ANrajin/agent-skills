# Discount Rules Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Discount requirements"` |
| Namespace (nopCommerce) | `Nop.Plugin.DiscountRules.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.DiscountRules.{Name}` |
| Core Interface | `IDiscountRequirementRule` |
| Class Naming | `{Name}DiscountRequirementRule` |

## Interface: `IDiscountRequirementRule`

Namespace: `Nop.Services.Discounts`

Inherits from `IPlugin`.

### Methods

| Method | Signature |
|--------|-----------|
| `CheckRequirementAsync` | `Task<DiscountRequirementValidationResult> CheckRequirementAsync(DiscountRequirementValidationRequest request)` |
| `GetConfigurationUrl` | `string GetConfigurationUrl(int discountId, int? discountRequirementId)` |

### Parameter/Result Types

`DiscountRequirementValidationRequest` contains:
- `DiscountRequirementId` — the specific requirement instance
- `Customer` — the customer being validated
- `Store` — the current store

`DiscountRequirementValidationResult` contains:
- `IsValid` — `bool`, whether the requirement is met
- `UserError` — optional error message to show the customer

## Plugin Class Template

```csharp
public class {Name}DiscountRequirementRule : BasePlugin, IDiscountRequirementRule
{
    // Inject: IDiscountService, ILocalizationService, INopUrlHelper, ISettingService

    public string GetConfigurationUrl(int discountId, int? discountRequirementId)
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration,
            new { discountId, discountRequirementId });
    }

    public override async Task InstallAsync()
    {
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        // Clean up discount requirements that use this rule
        var requirements = (await _discountService.GetAllDiscountRequirementsAsync())
            .Where(r => r.DiscountRequirementRuleSystemName == {Name}Defaults.SystemName)
            .ToList();

        foreach (var requirement in requirements)
            await _discountService.DeleteDiscountRequirementAsync(requirement);

        await _localizationService.DeleteLocaleResourcesAsync("Plugins.DiscountRules.{Name}");
        await base.UninstallAsync();
    }

    // --- IDiscountRequirementRule members ---

    public async Task<DiscountRequirementValidationResult> CheckRequirementAsync(DiscountRequirementValidationRequest request)
    {
        var result = new DiscountRequirementValidationResult();

        // Load settings for this specific requirement
        var settings = await _settingService.LoadSettingAsync<{Name}Settings>(
            key: $"{Name}Defaults.SettingsKey}-{request.DiscountRequirementId}");

        // Check if the customer meets the requirement
        result.IsValid = await EvaluateRequirement(request.Customer, settings);

        return result;
    }
}
```

## Unique Architecture

- **Settings stored per-discount-requirement**, not globally: Discount rule settings are stored with a key per requirement (e.g., `DiscountRequirement.MustBeInCustomerRole-{discountRequirementId}`), NOT via a shared `ISettings` class.
- **No dedicated settings class** for global settings — each discount requirement instance has its own settings saved with a unique key via `ISettingService`.
- **Admin controller uses `BasePluginController`** (not `BaseAdminController`): Controller still carries `[AuthorizeAdmin]`, `[Area(AreaNames.ADMIN)]` attributes.
- **Inline AJAX configuration**: The `Configure` action returns a partial view (sets `Layout = ""` in the view) rendered inline within the admin discount requirements tab — not a full admin page.
- **POST action returns JSON**: After saving, the POST action returns `Ok(new { NewRequirementId = ... })` for AJAX handling.
- **Model**: Simple model class with the discount-specific parameters plus `DiscountId` and `RequirementId` as hidden fields.
- **RouteProvider**: Implements `IRouteProvider` to register the admin configure route.
- **Event consumer** (recommended): Implement `IConsumer<EntityDeletedEvent<DiscountRequirement>>` to clean up orphaned settings when a discount requirement entity is deleted.
- **No ViewComponents**: Discount rules have no public UI rendering.
- **Permissions**: Use `StandardPermission.Promotions.DISCOUNTS_VIEW` and `StandardPermission.Promotions.DISCOUNTS_CREATE_EDIT_DELETE` (not custom permissions).
- **Required files**: `{Name}DiscountRequirementRule.cs`, `{Name}Defaults.cs`, `Controllers/DiscountRules{Name}Controller.cs`, `Models/RequirementModel.cs`, `Validators/RequirementModelValidator.cs`, `Infrastructure/RouteProvider.cs`, `Events/DiscountRequirementEventConsumer.cs`

## Install/Uninstall Checklist

**Install:**
- Add localized resources
- No settings to save (settings are per-requirement)

**Uninstall:**
- **Delete all discount requirements** that reference this rule — critical to avoid orphaned data
- Delete localized resources

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Areas/Admin/Controllers/DiscountRules{Name}Controller.cs` | Yes |
| `Areas/Admin/Views/DiscountRules{Name}/Configure.cshtml` | Yes (partial view, no layout) |
| `Areas/Admin/Models/` | Yes (requirement model) |
| `Areas/Admin/Validators/` | Yes |
| `Infrastructure/RouteProvider.cs` | Yes |
| `Events/` | Yes (EntityDeletedEvent consumer) |
| `Settings/` | No (settings are per-requirement, not a global class) |
| `Controllers/` (public) | No |
| `Components/` | No |
| `Domain/` | No |
| `Services/` | No (logic is in the rule class itself) |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BasePluginController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Do not create a global settings class** — discount rules store settings per-requirement with a composite key
- The `GetConfigurationUrl` method takes `(int discountId, int? discountRequirementId)` — the URL must include both as query parameters
- Configure view MUST set `Layout = ""` — it renders as a partial inside the discount edit tab
- POST action must return JSON (success with `NewRequirementId` or errors) — not a redirect
- The `CheckRequirementAsync` result is `DiscountRequirementValidationResult`, not a raw boolean — set both `IsValid` and optionally `UserError`
- Event consumer for `EntityDeletedEvent<DiscountRequirement>` is strongly recommended to clean up settings when admin deletes a requirement
