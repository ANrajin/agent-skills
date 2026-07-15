# Payment Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Payment methods"` |
| Namespace (nopCommerce) | `Nop.Plugin.Payments.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.Payment.{Name}` |
| Core Interface | `IPaymentMethod` |
| Class Naming | `{Name}PaymentMethod` or `{Name}PaymentProcessor` |

## Interface: `IPaymentMethod`

Namespace: `Nop.Services.Payments`

### Methods

| Method | Signature |
|--------|-----------|
| `ProcessPaymentAsync` | `Task<ProcessPaymentResult> ProcessPaymentAsync(ProcessPaymentRequest processPaymentRequest)` |
| `PostProcessPaymentAsync` | `Task PostProcessPaymentAsync(PostProcessPaymentRequest postProcessPaymentRequest)` |
| `HidePaymentMethodAsync` | `Task<bool> HidePaymentMethodAsync(IList<ShoppingCartItem> cart)` |
| `GetAdditionalHandlingFeeAsync` | `Task<decimal> GetAdditionalHandlingFeeAsync(IList<ShoppingCartItem> cart)` |
| `CaptureAsync` | `Task<CapturePaymentResult> CaptureAsync(CapturePaymentRequest capturePaymentRequest)` |
| `RefundAsync` | `Task<RefundPaymentResult> RefundAsync(RefundPaymentRequest refundPaymentRequest)` |
| `VoidAsync` | `Task<VoidPaymentResult> VoidAsync(VoidPaymentRequest voidPaymentRequest)` |
| `ProcessRecurringPaymentAsync` | `Task<ProcessPaymentResult> ProcessRecurringPaymentAsync(ProcessPaymentRequest processPaymentRequest)` |
| `CancelRecurringPaymentAsync` | `Task<CancelRecurringPaymentResult> CancelRecurringPaymentAsync(CancelRecurringPaymentRequest cancelRecurringPaymentRequest)` |
| `CanRePostProcessPaymentAsync` | `Task<bool> CanRePostProcessPaymentAsync(Order order)` |
| `ValidatePaymentFormAsync` | `Task<IList<string>> ValidatePaymentFormAsync(IFormCollection form)` |
| `GetPaymentInfoAsync` | `Task<ProcessPaymentRequest> GetPaymentInfoAsync(IFormCollection form)` |
| `GetPublicViewComponent` | `Type GetPublicViewComponent()` |
| `GetPaymentMethodDescriptionAsync` | `Task<string> GetPaymentMethodDescriptionAsync()` |

### Properties

| Property | Type | Purpose |
|----------|------|---------|
| `SupportCapture` | `bool` | Whether capture is supported |
| `SupportPartiallyRefund` | `bool` | Partial refund support |
| `SupportRefund` | `bool` | Full refund support |
| `SupportVoid` | `bool` | Void support |
| `RecurringPaymentType` | `RecurringPaymentType` | `NotSupported` / `Manual` / `Automatic` |
| `PaymentMethodType` | `PaymentMethodType` | `Standard` / `Redirection` / `Button` / `StandardAndButton` |
| `SkipPaymentInfo` | `bool` | Whether to skip payment info page |

## Plugin Class Template

```csharp
public class {Name}PaymentMethod : BasePlugin, IPaymentMethod
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService, ILogger, IPermissionService

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
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.Payments.{Name}");
        await base.UninstallAsync();
    }

    // --- IPaymentMethod members ---
    // Gating methods (return false by default for non-applicable gateways)
    public Task<bool> HidePaymentMethodAsync(IList<ShoppingCartItem> cart) => Task.FromResult(false);
    public Task<decimal> GetAdditionalHandlingFeeAsync(IList<ShoppingCartItem> cart) => Task.FromResult(decimal.Zero);

    public Task<bool> CanRePostProcessPaymentAsync(Order order) => Task.FromResult(false);

    // Capture/Refund/Void (implement based on gateway capability)
    public Task<CapturePaymentResult> CaptureAsync(CapturePaymentRequest request) => ...;
    public Task<RefundPaymentResult> RefundAsync(RefundPaymentRequest request) => ...;
    public Task<VoidPaymentResult> VoidAsync(VoidPaymentRequest request) => ...;

    // Recurring (return unsupported result if not applicable)
    public Task<ProcessPaymentResult> ProcessRecurringPaymentAsync(ProcessPaymentRequest request) => ...;
    public Task<CancelRecurringPaymentResult> CancelRecurringPaymentAsync(CancelRecurringPaymentRequest request) => ...;

    // Main processing
    public Task<ProcessPaymentResult> ProcessPaymentAsync(ProcessPaymentRequest request) => ...;
    public Task PostProcessPaymentAsync(PostProcessPaymentRequest request) => Task.CompletedTask;

    // Form handling
    public Task<IList<string>> ValidatePaymentFormAsync(IFormCollection form) => Task.FromResult<IList<string>>(new List<string>());
    public Task<ProcessPaymentRequest> GetPaymentInfoAsync(IFormCollection form) => ...;

    // ViewComponent for checkout
    public Type GetPublicViewComponent() => typeof({Name}PaymentViewComponent);

    public async Task<string> GetPaymentMethodDescriptionAsync() => ...;

    // Properties
    public bool SupportCapture => {true/false};
    public bool SupportPartiallyRefund => {true/false};
    public bool SupportRefund => {true/false};
    public bool SupportVoid => {true/false};
    public RecurringPaymentType RecurringPaymentType => RecurringPaymentType.NotSupported;
    public PaymentMethodType PaymentMethodType => PaymentMethodType.{Standard/Redirection/Button};
    public bool SkipPaymentInfo => false;
}
```

## Unique Architecture

- **ViewComponent for checkout form**: `GetPublicViewComponent()` returns a `Type` of ViewComponent that renders payment form on the checkout page. No public controller for standard checkout flow.
- **PaymentMethodType determines behavior**:
  - `Standard` — payment form collected on-site, processed via AJAX or post-payment
  - `Redirection` — customer redirected to external gateway page (e.g., PayPal), `PostProcessPaymentAsync` handles the redirect
  - `Button` — express checkout button rendered in cart (e.g., PayPal Pay Now)
- **Admin controller**: Named `{Name}Controller`, inherits `BaseAdminController`, typically has Configure GET/POST for settings
- **No public controllers** needed — the ViewComponent + payment flow replaces them
- **Required files**: `{Name}PaymentMethod.cs`, `{Name}Defaults.cs`, `{Name}Settings.cs`, `Components/{Name}PaymentViewComponent.cs`, `Areas/Admin/Controllers/{Name}Controller.cs`, `Views/Shared/Components/{Name}Payment/Default.cshtml`

## Install/Uninstall Checklist

**Install:**
- Save default settings via `_settingService.SaveSettingAsync()`
- Add localized resources
- Optionally register scheduled task for transaction status polling

**Uninstall:**
- Delete settings via `_settingService.DeleteSettingAsync<>()`
- Delete localized resources
- Remove any scheduled tasks
- Do NOT delete order records

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Components/{Name}PaymentViewComponent.cs` | Yes (checkout form) |
| `Areas/Admin/Controllers/` | Yes (settings page) |
| `Controllers/` (public) | No (except for special callback URLs) |
| `Domain/` | Only if custom entities (e.g., PayPal stored payment methods) |
| `Data/` | Only if custom entities |
| `Factories/` | Only if complex admin model prep needed |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` instead of `BaseAdminController`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Define permissions via `IPermissionConfigManager`
- Admin menu via `AdminMenuCreatedEventConsumer` with `NopStationAdminMenuItem`
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Never** put the payment form rendering in a controller action — always use a ViewComponent returned by `GetPublicViewComponent()`
- `PaymentMethodType.Standard` mandates `ValidatePaymentFormAsync` and `GetPaymentInfoAsync` implementation
- `PaymentMethodType.Redirection` mandates `PostProcessPaymentAsync` implementation
- `ProcessPaymentRequest` contains `OrderGuid`, `OrderTotal`, `CustomerId`, `StoreId` — do not trust these values blindly for security-sensitive gateways
- `PostProcessPaymentRequest` contains the newly created `Order` object — use it for redirect URL construction
- For webhook/callback handling, create a dedicated public controller in `Controllers/` (not in admin area)
