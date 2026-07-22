# External Authentication Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"ExternalAuth methods"` |
| Namespace (nopCommerce) | `Nop.Plugin.ExternalAuth.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.ExternalAuth.{Name}` |
| Core Interface | `IExternalAuthenticationMethod` |
| Class Naming | `{Name}AuthenticationMethod` |

## Interface: `IExternalAuthenticationMethod`

Namespace: `Nop.Services.Authentication.External`

Inherits from `IPlugin`.

### Methods

| Method | Signature |
|--------|-----------|
| `GetPublicViewComponent` | `Type GetPublicViewComponent()` |

Returns the `Type` of a ViewComponent that renders the external login button (e.g., "Sign in with Facebook", "Sign in with Google") on the public login page.

## Plugin Class Template

```csharp
public class {Name}AuthenticationMethod : BasePlugin, IExternalAuthenticationMethod
{
    // Inject: INopUrlHelper, ISettingService, ILocalizationService

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}ExternalAuthSettings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}ExternalAuthSettings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.ExternalAuth.{Name}");
        await base.UninstallAsync();
    }

    // --- IExternalAuthenticationMethod members ---

    public Type GetPublicViewComponent() => typeof({Name}AuthenticationViewComponent);
}
```

## Unique Architecture

- **Critical component — `IExternalAuthenticationRegistrar`**: ExternalAuth plugins require a **registrar** class that implements `IExternalAuthenticationRegistrar.Configure(AuthenticationBuilder builder)` to register the OAuth middleware with ASP.NET Core's authentication stack (e.g., `builder.AddFacebook()`). Without this, the OAuth flow won't work.
- **Public controller**: Unlike most plugin types, ExternalAuth plugins require a **public controller** with:
  - `Login` action — initiates OAuth challenge: `return Challenge(authenticationProperties, "{SchemeName}");`
  - `LoginCallback` action — handles OAuth callback, creates `ExternalAuthenticationParameters`, calls `_externalAuthenticationService.AuthenticateAsync()`
  - Optional GDPR data deletion endpoints (for providers like Facebook that require it)
- **ViewComponent for login button**: `GetPublicViewComponent()` returns a ViewComponent that renders the "Sign in with X" button. The ViewComponent inherits from `NopViewComponent`.
- **Settings**: Stores OAuth client credentials (`ClientKeyIdentifier`, `ClientSecret`).
- **RouteProvider**: Registers public routes for data deletion callbacks (not admin routes).
- **Event consumer** (optional): `CustomerAutoRegisteredByExternalMethodEvent` to copy claims (name, avatar) from the external provider to the customer profile.
- **Admin controller**: Named `{Name}AuthenticationController`, inherits `BasePluginController` with `[AuthorizeAdmin]`, `[Area(AreaNames.ADMIN)]`, and `[CheckPermission(StandardPermission.Configuration.MANAGE_EXTERNAL_AUTHENTICATION_METHODS)]`.
- **Permissions**: Use `StandardPermission.Configuration.MANAGE_EXTERNAL_AUTHENTICATION_METHODS` (not custom).
- **Required files**: `{Name}AuthenticationMethod.cs`, `{Name}Defaults.cs`, `{Name}ExternalAuthSettings.cs`, `Services/{Name}AuthenticationRegistrar.cs` (implements `IExternalAuthenticationRegistrar`), `Components/{Name}AuthenticationViewComponent.cs`, `Controllers/{Name}AuthenticationController.cs` (public + admin), `Infrastructure/RouteProvider.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings (ClientId, ClientSecret)
- Add localized resources

**Uninstall:**
- Delete settings
- Delete localized resources

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Components/{Name}AuthenticationViewComponent.cs` | Yes (login button) |
| `Controllers/{Name}AuthenticationController.cs` | Yes (public: Login, LoginCallback; admin: Configure) |
| `Controllers/` (public-only) | Yes (login/callback actions) |
| `Areas/Admin/Controllers/` | Optional (configure can be in same controller with area attributes) |
| `Services/{Name}AuthenticationRegistrar.cs` | Yes (OAuth middleware registration) |
| `Infrastructure/RouteProvider.cs` | Yes (for callback routes) |
| `Events/` | Only if handling auto-registration events |
| `Domain/` | No |
| `Data/` | No |
| `Views/` (public) | Yes (ViewComponent view + optionally login button styles) |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `NopStationAdminController` for admin actions
- Use `NopStationPublicController` for public actions
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings

## Guardrails

- **Must implement `IExternalAuthenticationRegistrar`** — missing this is the most common error. Without it, `builder.AddFacebook()` (or equivalent) is never called and the OAuth middleware is not registered.
- The registrar is a separate class registered via `INopStartup`, not part of the plugin class
- The `Login` action must call `HttpContext.ChallengeAsync()` or `return Challenge(properties, scheme)` — not a redirect
- `LoginCallback` must handle both success (redirect to return URL) and failure (redirect to login page with error)
- The ViewComponent for the login button is rendered in the public login page — it should be minimal (just a button/link)
- GDPR data deletion is required by Facebook and some other providers — implement it even if the provider doesn't require it yet
- The OAuth authentication scheme name must match between registrar configuration and controller challenge calls
