# Exchange Rate Provider Plugin Scaffold

## Plugin Identity

| Field | Value |
|---|---|
| `plugin.json` Group | `"Exchange rate providers"` |
| Namespace (nopCommerce) | `Nop.Plugin.ExchangeRate.{Name}` |
| Namespace (NopStation) | `NopStation.Plugin.ExchangeRate.{Name}` |
| Core Interface | `IExchangeRateProvider` |
| Class Naming | `{Name}ExchangeRateProvider` |

## Interface: `IExchangeRateProvider`

Namespace: `Nop.Services.Directory`

### Methods

| Method | Signature |
|--------|-----------|
| `GetCurrencyLiveRatesAsync` | `Task<IList<Core.Domain.Directory.ExchangeRate>> GetCurrencyLiveRatesAsync(string exchangeRateCurrencyCode)` |

### Return Type

Returns `IList<Core.Domain.Directory.ExchangeRate>` where `ExchangeRate` contains:
- `CurrencyCode` — three-letter ISO code (e.g., "USD", "EUR")
- `Rate` — decimal exchange rate
- `UpdatedOnUtc` — when the rate was last updated

## Plugin Class Template

```csharp
public class {Name}ExchangeRateProvider : BasePlugin, IExchangeRateProvider
{
    // Inject: IHttpClientFactory, ISettingService, ILocalizationService

    // No config page — exchange rate providers are configured inline in Admin > Currency
    // public override string GetConfigurationPageUrl() not needed

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new {Name}ExchangeRateSettings());
        await _localizationService.AddOrUpdateLocaleResourceAsync(GetLocalizableStrings());
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        await _settingService.DeleteSettingAsync<{Name}ExchangeRateSettings>();
        await _localizationService.DeleteLocaleResourcesAsync("Plugins.ExchangeRate.{Name}");
        await base.UninstallAsync();
    }

    // --- IExchangeRateProvider members ---

    public async Task<IList<Core.Domain.Directory.ExchangeRate>> GetCurrencyLiveRatesAsync(string exchangeRateCurrencyCode)
    {
        // Fetch XML/JSON from external source (e.g., ECB, Yahoo Finance, Open Exchange Rates)
        // Parse the response and return exchange rates relative to the given currency code

        var rates = new List<Core.Domain.Directory.ExchangeRate>();

        using var httpClient = _httpClientFactory.CreateClient({Name}Defaults.HttpClientName);
        var response = await httpClient.GetStringAsync(_settings.ApiUrl);

        // Parse response
        rates.Add(new Core.Domain.Directory.ExchangeRate
        {
            CurrencyCode = "USD",
            Rate = 1.1234M,
            UpdatedOnUtc = DateTime.UtcNow
        });

        return rates;
    }
}
```

## Unique Architecture

- **Simplest plugin type** — only 1 method to implement
- **No admin controller**: Exchange rate providers have no configuration page URL. Configuration is done inline in nopCommerce Admin > Currency settings page, which shows all installed exchange rate providers as a dropdown.
- **No `GetConfigurationPageUrl()` override** — the plugin class does NOT override this method (or returns empty/null)
- **No admin Views/Controllers**: None needed. The only UI nopCommerce provides is built into the Currency list page.
- **Domain entity**: Uses nopCommerce's built-in `Core.Domain.Directory.ExchangeRate` — no custom domain entity needed
- **Required files**: `{Name}ExchangeRateProvider.cs`, `{Name}Defaults.cs`, `{Name}ExchangeRateSettings.cs`

## Install/Uninstall Checklist

**Install:**
- Save default settings (API endpoint URL, API key)
- Add localized resources

**Uninstall:**
- Delete settings
- Delete localized resources

## Files to Include vs Exclude

| File/Directory | Required |
|---------------|----------|
| `Areas/Admin/Controllers/` | No |
| `Areas/Admin/Views/` | No |
| `Areas/Admin/Models/` | No |
| `Controllers/` (public) | No |
| `Components/` | No |
| `Domain/` | No |
| `Data/` | No |
| `Services/` | Usually not needed (logic is in the provider class) |

## NopStation Variant

When this is a NopStation plugin, also apply the `nopstation-plugin.md` rule:
- Plugin class additionally implements `INopStationPlugin`
- Use `this.InstallPluginAsync()` / `this.UninstallPluginAsync()` for lifecycle
- Implement `GetPluginResources()` for localized strings
- Still no admin controller needed (even for NopStation)

## Guardrails

- **No admin controller** — this is the only plugin type that consistently has no admin UI. Do not scaffold an admin controller unless explicitly required.
- Use `IHttpClientFactory` for HTTP requests — never create `HttpClient` directly
- The `exchangeRateCurrencyCode` parameter is the base currency code — all returned rates should be relative to this currency
- Return rates for multiple currencies, not just one — nopCommerce expects a list
- Handle API errors gracefully — `GetCurrencyLiveRatesAsync` is called on a schedule, errors should be logged but not crash the task
- Settings are minimal — usually just an API URL and optional key
