# Multi-Factor Authentication Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Multi-factor authentication"` |
| Namespace (nopCommerce) | `Nop.Plugin.MultiFactorAuth.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.MultiFactorAuth.{Name}` |
| Core Interface | `IMultiFactorAuthenticationMethod` |
| Class Naming | `{Name}Method` or `{Name}MultiFactorAuthenticationMethod` |

## Interface: `IMultiFactorAuthenticationMethod`

Namespace: `Nop.Services.Authentication.MultiFactor`

Inherits from `IPlugin`.

### Members

| Member | Type | Signature |
|--------|------|-----------|
| `Type` | Property | `MultiFactorAuthenticationType Type { get; }` |
| `GetPublicViewComponent` | Method | `Type GetPublicViewComponent()` |
| `GetVerificationViewComponent` | Method | `Type GetVerificationViewComponent()` |
| `GetDescriptionAsync` | Method | `Task<string> GetDescriptionAsync()` |

### MultiFactorAuthenticationType Enum

```csharp
public enum MultiFactorAuthenticationType
{
    ApplicationVerification = 0,    // TOTP apps (Google/Microsoft Authenticator)
    SMSVerification = 1,
    EmailVerification = 2,
}
```

## Plugin Class Template

```csharp
public class {Name}Method : BasePlugin, IMultiFactorAuthenticationMethod
{
    // Inject: INopUrlHelper, ILocalizationService, ISettingService, IStoreContext

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
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.MultiFactorAuth.{Name}");
        await base.UninstallAsync();
    }

    // --- IMultiFactorAuthenticationMethod members ---

    public MultiFactorAuthenticationType Type => MultiFactorAuthenticationType.ApplicationVerification;

    public Type GetPublicViewComponent() => typeof({Name}SetupViewComponent);

    public Type GetVerificationViewComponent() => typeof({Name}VerificationViewComponent);

    public async Task<string> GetDescriptionAsync()
    {
        return await _localizationService.GetResourceAsync("Plugins.MultiFactorAuth.{Name}.Description");
    }
}
```

## Unique Architecture

- **Two ViewComponents required**:
  1. **Setup ViewComponent** (`GetPublicViewComponent`) — rendered in customer account area, allows customers to configure MFA (scan QR code, enter secret key, test token)
  2. **Verification ViewComponent** (`GetVerificationViewComponent`) — rendered during login, presents a token input field (e.g., 6-digit TOTP code)
- **Domain entity** for storing MFA configurations per customer (e.g., `{Name}Record : BaseEntity` with Customer, SecretKey, IsActive fields)
- **Schema migration** required to create the MFA configuration table
- **TOTP service**: Typically uses a NuGet package (e.g., Google Authenticator) for TOTP generation and validation. The service handles:
  - Generating shared secrets
  - Generating QR codes for authenticator app setup
  - Validating time-based tokens
- **Two public controllers**:
  1. `{Name}Controller` (admin) — settings page with datagrid of registered customer configurations
  2. `AuthenticationController` (public) — Register/Verify actions for customer-side MFA flow
- **Model factory**: Prepares setup models (QR code data, manual entry key) for the setup ViewComponent
- **RouteProvider**: Registers admin configure route
- **Permissions**: Use `StandardPermission.Configuration.MANAGE_MULTIFACTOR_AUTHENTICATION_METHODS` (not custom)
- **Validators**: Validate token input (6-digit numeric code)
- **Required files**: `{Name}Method.cs`, `{Name}Defaults.cs`, `{Name}Settings.cs`, `Domain/{Name}Record.cs`, `Data/SchemaMigration.cs`, `Services/I{Name}Service.cs`, `Services/{Name}Service.cs`, `Factories/I{Name}ModelFactory.cs`, `Factories/{Name}ModelFactory.cs`, `Components/{Name}SetupViewComponent.cs`, `Components/{Name}VerificationViewComponent.cs`, `Controllers/{Name}Controller.cs` (admin), `Controllers/AuthenticationController.cs` (public), `Infrastructure/RouteProvider.cs`, `Infrastructure/NopStartup.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings (QR code size, business prefix)
- Install schema migration for customer MFA records table
- Add localized resources

**Uninstall:**
- Delete settings
- Delete localized resources
- Drop MFA configuration table (or keep — depends on business preference)

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Components/{Name}SetupViewComponent.cs` | Yes (MFA setup) |
| `Components/{Name}VerificationViewComponent.cs` | Yes (login verification) |
| `Controllers/{Name}Controller.cs` | Yes (admin: settings + config grid) |
| `Controllers/AuthenticationController.cs` | Yes (public: register + verify) |
| `Domain/{Name}Record.cs` | Yes (MFA config entity) |
| `Data/SchemaMigration.cs` | Yes |
| `Services/I{Name}Service.cs` + `Services/{Name}Service.cs` | Yes (TOTP logic, data access) |
| `Factories/` | Yes (model factory for setup ViewComponent) |
| `Infrastructure/NopStartup.cs` | Yes (DI registration) |
| `Infrastructure/RouteProvider.cs` | Yes |
| `Validators/` | Yes (token validation) |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` for admin controller
- Use `NopStationPublicController` for public controllers
- Use `NopStationViewComponent` for ViewComponents
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Two ViewComponents are mandatory** — one for setup (customer account) and one for verification (login). Missing either makes the plugin non-functional.
- The `Type` property determines the MFA method category (app, SMS, email) — set it correctly
- Store secrets **encrypted** in the database — TOTP secrets are sensitive credentials
- The verification ViewComponent is rendered during the login flow — it must be simple and fast (no external API calls)
- Validation of the 6-digit TOTP code must handle time drift (typically ±1 interval is allowed)
- Schema migration is almost always needed to store customer MFA configurations
- Model factories should prepare the QR code as a data URL or base64 image string for the setup view
