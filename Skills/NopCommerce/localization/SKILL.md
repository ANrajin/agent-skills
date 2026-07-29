---
name: localization
description: >-
  Implement nopCommerce plugin localization end-to-end: defining resource
  keys in GetPluginResources(), binding them via [NopResourceDisplayName],
  using @T() in Razor views, wiring FluentValidation error messages,
  controller success/error notifications, localized service/task logging,
  localized enum display names, and admin menu titles. Use whenever a plugin
  adds or touches any user-facing string, not only when first defining
  resource keys.
---

# Localization Skill

## When to Use

Activate this skill when:
- Defining new localized string resources in a plugin's `GetPluginResources()` method
- Adding `[NopResourceDisplayName]` attributes to model properties
- Using `@T()` in Razor views (page titles, card headers, grid columns, buttons, alerts, modals, JavaScript)
- Writing FluentValidation messages with `ILocalizationService`
- Displaying success/error notifications in admin controllers
- Returning localized API error responses
- Logging localized messages in services or scheduled tasks
- Localizing enum values for dropdown display
- Building admin menu items with localized titles

## How to Use

### Step 1 — Define Resource Keys in the Plugin Class

All resource strings live in `GetPluginResources()` inside the `{PluginName}Plugin.cs` file. Resources are installed via `_localizationService.AddOrUpdateLocaleResourceAsync(GetPluginResources())` during `InstallAsync()` and `UpdateAsync()`.

```csharp
public IDictionary<string, string> GetPluginResources()
{
    return new Dictionary<string, string>
    {
        ["Plugin.{PluginName}.Title"] = "My Plugin",
        ["Plugin.{PluginName}.Admin.Menu.Title"] = "My Plugin",
        ["Plugin.{PluginName}.Admin.Menu.Configuration"] = "Configuration",

        ["Plugin.{PluginName}.Configuration.Title"] = "My Plugin configuration",
        ["Plugin.{PluginName}.Configuration.Updated"] = "Configuration saved successfully.",
        ["Plugin.{PluginName}.Configuration.GeneralSettings"] = "General settings",

        ["Plugin.{PluginName}.Fields.Enabled"] = "Enabled",
        ["Plugin.{PluginName}.Fields.Enabled.Hint"] = "Enable or disable the plugin.",

        ["Plugin.{PluginName}.Enums.MyEnum.ValueOne"] = "Value One",
        ["Plugin.{PluginName}.Enums.MyEnum.ValueTwo"] = "Value Two",

        ["Plugin.{PluginName}.{ModelName}.Name.Required"] = "Name is required.",
        ["Plugin.{PluginName}.Api.InvalidApiKey"] = "Invalid API key.",
        ["Plugin.{PluginName}.Task.ExecutionError"] = "Error in task execution.",
        ["Plugin.{PluginName}.UnexpectedError"] = "An unexpected error occurred."
    };
}
```

Uninstall must clean up by prefix:

```csharp
public override async Task UninstallAsync()
{
    await _localizationService.DeleteLocaleResourcesAsync("Plugin.{PluginName}");
    await base.UninstallAsync();
}
```

### Step 2 — Bind to Models with `[NopResourceDisplayName]`

The `[NopResourceDisplayName]` attribute auto-resolves `<key>` for the label and `<key>.Hint` for the tooltip. Always define both in `GetPluginResources()`.

```csharp
[NopResourceDisplayName("Plugin.{PluginName}.Fields.Enabled")]
public bool Enabled { get; set; }

[NopResourceDisplayName("Plugin.{PluginName}.Fields.MaxRetries")]
public int MaxRetries { get; set; }
```

This powers `<nop-label asp-for="Enabled" />` in views — the label text and hint tooltip are rendered automatically.

### Step 3 — Use `@T()` in Razor Views

#### Page Titles

```cshtml
@{
    ViewBag.PageTitle = T("Plugin.{PluginName}.Configuration.Title").Text;
}
```

#### Content Headers

```cshtml
<h1 class="float-left">
    @T("Plugin.{PluginName}.Title") - @T("Plugin.{PluginName}.Admin.Menu.Configuration")
</h1>
```

#### nop-card Titles

```cshtml
<nop-card asp-name="general-settings"
          asp-icon="fas fa-gear"
          asp-title="@T("Plugin.{PluginName}.Configuration.GeneralSettings")"
          asp-hide-block-attribute-name="@hideGeneralBlockAttributeName"
          asp-hide="@hideGeneralBlock"
          asp-advanced="false">
    @await Html.PartialAsync("_Configure.GeneralSettings", Model)
</nop-card>
```

#### DataTables Column Headers

```cshtml
new ColumnProperty(nameof(Model.Name))
{
    Title = T("Plugin.{PluginName}.Entity.Fields.Name").Text,
    Width = "200"
}
```

> **Important:** Use `.Text` when assigning to a `string` property (e.g., `Title`). Use `@T(...)` directly in HTML markup.

#### Buttons and Labels

```cshtml
<button type="submit" class="btn btn-primary">
    <i class="far fa-save"></i>
    @T("Admin.Common.Save")
</button>
```

#### Modals

```cshtml
<h4 class="modal-title">@T("Plugin.{PluginName}.Modal.Title")</h4>
```

#### JavaScript Blocks Inside Razor

```cshtml
<script asp-location="Footer">
    $.ajax({
        error: function () {
            $('#error-el').text('@T("Plugin.{PluginName}.UnexpectedError")').show();
        }
    });
</script>
```

### Step 4 — Localize FluentValidation Messages

#### Admin Validators (no RuleSet)

```csharp
public class ConfigurationValidator : BaseNopValidator<ConfigurationModel>
{
    public ConfigurationValidator(ILocalizationService localizationService)
    {
        RuleFor(x => x.MaxRetries)
            .GreaterThan(0)
            .When(x => x.Enabled)
            .WithMessageAwait(localizationService.GetResourceAsync(
                "Plugin.{PluginName}.Configuration.MaxRetries.Required"));
    }
}
```

#### API Validators (with RuleSet)

```csharp
public partial class ItemRequestValidator : BaseNopValidator<ItemRequest>
{
    public ItemRequestValidator(ILocalizationService localizationService)
    {
        RuleSet(NopValidationDefaults.ValidationRuleSet, () =>
        {
            RuleFor(x => x.Name)
                .NotEmpty()
                .WithMessageAwait(localizationService
                    .GetResourceAsync("Plugin.{PluginName}.ItemRequest.Name.Required"));
        });
    }
}
```

### Step 5 — Controller Notifications

```csharp
_notificationService.SuccessNotification(
    await _localizationService.GetResourceAsync("Plugin.{PluginName}.Configuration.Updated"));
```

### Step 6 — API Error Responses

Create a utility method in API controllers to return localized errors:

```csharp
private async Task<IActionResult> ErrorAsync(int statusCode, string resourceKey)
{
    var message = await _localizationService.GetResourceAsync(resourceKey);
    return StatusCode(statusCode, new ApiErrorResponse { Error = message });
}

// Usage
return await ErrorAsync(
    (int)HttpStatusCode.BadRequest,
    "Plugin.{PluginName}.Api.PluginDisabled");
```

### Step 7 — Localized Logging in Services and Scheduled Tasks

Use `ILocalizationService` + `ILogger` from `Nop.Services.Logging`:

```csharp
await _logger.InformationAsync(
    string.Format(
        await _localizationService.GetResourceAsync("Plugin.{PluginName}.Task.ItemProcessed"),
        itemId));

await _logger.ErrorAsync(
    await _localizationService.GetResourceAsync("Plugin.{PluginName}.Task.ExecutionError"), ex);
```

Use `string.Format()` with `{0}`, `{1}` placeholders for parameterized messages:

```csharp
["Plugin.{PluginName}.Task.ItemProcessed"] = "Item {0} processed successfully.",
["Plugin.{PluginName}.Task.ItemFailed"] = "Item {0} failed: {1}.",
```

### Step 8 — Localize Enum Display Names

```csharp
["Plugin.{PluginName}.Enums.SyncStatus.Pending"] = "Pending",
["Plugin.{PluginName}.Enums.SyncStatus.Synced"] = "Synced",
["Plugin.{PluginName}.Enums.SyncStatus.Failed"] = "Failed",
```

Use `ILocalizationService` to resolve these when building `SelectListItem` dropdowns in model factories.

### Step 9 — Admin Menu Localization

```csharp
var rootMenu = new AdminMenuItem
{
    Title = await _localizationService.GetResourceAsync("Plugin.{PluginName}.Admin.Menu.Title"),
    // ...
};

rootMenu.ChildNodes.Add(new AdminMenuItem
{
    Title = await _localizationService.GetResourceAsync("Plugin.{PluginName}.Admin.Menu.Configuration"),
    // ...
});
```

## Common Patterns

### Resource Key Naming Convention

| Category | Pattern | Example |
|----------|---------|---------|
| Plugin title | `Plugin.{Name}.Title` | `Plugin.BusinessCentralConnector.Title` |
| Admin menu | `Plugin.{Name}.Admin.Menu.{Item}` | `Plugin.BusinessCentralConnector.Admin.Menu.Configuration` |
| Page titles | `Plugin.{Name}.Admin.{Area}.Title` | `Plugin.BusinessCentralConnector.Admin.SyncSessions.Title` |
| Config section | `Plugin.{Name}.Configuration.{Section}` | `Plugin.BusinessCentralConnector.Configuration.GeneralSettings` |
| Config notifications | `Plugin.{Name}.Configuration.Updated` | `Plugin.BusinessCentralConnector.Configuration.Updated` |
| Field labels | `Plugin.{Name}.Fields.{Property}` | `Plugin.BusinessCentralConnector.Fields.Enabled` |
| Field hints | `Plugin.{Name}.Fields.{Property}.Hint` | `Plugin.BusinessCentralConnector.Fields.Enabled.Hint` |
| Entity fields | `Plugin.{Name}.{Entity}.Fields.{Property}` | `Plugin.BusinessCentralConnector.SyncSessions.Fields.SessionId` |
| Search fields | `Plugin.{Name}.Fields.Search{Property}` | `Plugin.BusinessCentralConnector.Fields.SearchStatus` |
| Enum values | `Plugin.{Name}.Enums.{EnumType}.{Value}` | `Plugin.BusinessCentralConnector.Enums.SyncSessionStatus.Complete` |
| Validation | `Plugin.{Name}.{Model}.{Property}.{Rule}` | `Plugin.BusinessCentralConnector.SessionStartRequest.SessionId.Required` |
| Config validation | `Plugin.{Name}.Configuration.{Property}.Required` | `Plugin.BusinessCentralConnector.Configuration.MaxAttempts.Required` |
| API responses | `Plugin.{Name}.Api.{Action}` | `Plugin.BusinessCentralConnector.Api.SessionStarted` |
| API errors | `Plugin.{Name}.Api.{ErrorType}` | `Plugin.BusinessCentralConnector.Api.InvalidApiKey` |
| Task messages | `Plugin.{Name}.{TaskName}.{Action}` | `Plugin.BusinessCentralConnector.PromotionTask.ProductPromoted` |
| UI messages | `Plugin.{Name}.{Feature}.{Message}` | `Plugin.BusinessCentralConnector.TokenManagement.NoToken` |
| Generic errors | `Plugin.{Name}.UnexpectedError` | `Plugin.BusinessCentralConnector.UnexpectedError` |

### Label + Hint Pair Pattern

Every `[NopResourceDisplayName]` field requires two resource entries:

```csharp
["Plugin.{Name}.Fields.MaxRetries"] = "Max Retry Attempts",
["Plugin.{Name}.Fields.MaxRetries.Hint"] = "Maximum number of retry attempts before the record is skipped.",
```

### Parameterized Messages

Use `string.Format()` with positional placeholders — never string interpolation:

```csharp
["Plugin.{Name}.Task.Processed"] = "Processing {0} pending records.",
["Plugin.{Name}.Task.Failed"] = "Item {0} failed: {1}.",

// Usage
string.Format(await _localizationService.GetResourceAsync("Plugin.{Name}.Task.Failed"), itemId, ex.Message);
```

### Reuse nopCommerce Built-in Resources

For common UI elements, reuse existing nopCommerce resources instead of defining new ones:

```cshtml
@T("Admin.Common.Save")
@T("Admin.Common.Search")
@T("Admin.Common.View")
@T("Admin.Common.Close")
@T("Admin.Common.Loading")
@T("Admin.Configuration.Plugins.Misc.BackToList")
```

### View File Structure

```
_ViewImports.cshtml          → inject ILocalizationService (via NopRazorPage<TModel> @T),
                                IGenericAttributeService, INopHtmlHelper, IWorkContext
Configure.cshtml             → main page: ViewBag.PageTitle, content header, nop-cards
_Configure.{Section}.cshtml  → partial views: nop-label, nop-editor, validation spans
{ListPage}.cshtml            → search panel + DataTables grid with T(...).Text columns
```

## Guardrails

1. **NEVER hardcode user-facing text** — always use `@T()` in views or `_localizationService.GetResourceAsync()` in C# code.
2. **NEVER hardcode log messages** — always resolve via `_localizationService.GetResourceAsync()` before passing to `_logger`.
3. **ALWAYS define both label and hint** — every `[NopResourceDisplayName("key")]` must have a matching `"key.Hint"` resource.
4. **ALWAYS use the plugin prefix** — resource keys must start with `Plugin.{PluginName}.` to avoid collisions and enable clean uninstall via `DeleteLocaleResourcesAsync("Plugin.{PluginName}")`.
5. **NEVER use `string.Empty` as a resource key** — every resource key must be meaningful and follow the naming convention.
6. **ALWAYS use `.Text` in C# string contexts** — when assigning `T(...)` to a property like `Title` in `ColumnProperty`, use `T("...").Text`. In HTML markup use `@T("...")` directly.
7. **ALWAYS use `string.Format()` for parameterized messages** — never use string interpolation (`$"..."`) with resource strings.
8. **ALWAYS add new resources to `GetPluginResources()`** — never define resources outside this method.
9. **ALWAYS update both `InstallAsync()` and `UpdateAsync()`** — both must call `_localizationService.AddOrUpdateLocaleResourceAsync(GetPluginResources())` to ensure new strings are available on upgrade.
10. **ALWAYS bump the plugin version** when adding new resource strings (bump `.{fix}`).
11. **NEVER use `ILogger<T>` from `Microsoft.Extensions.Logging`** — use `ILogger` from `Nop.Services.Logging`.
12. **ALWAYS use `asp-location="Footer"` for script tags** — scripts in cshtml files must use `<script asp-location="Footer">`.
13. **Reuse nopCommerce built-in resources** — use `Admin.Common.Save`, `Admin.Common.Search`, etc. instead of creating plugin-specific duplicates.
14. **ALWAYS use `WithMessageAwait`** in FluentValidation — use `.WithMessageAwait(localizationService.GetResourceAsync("..."))` for async localized validation messages.
15. All local keys must follow the existing local keys suffix. If not defined, ask the user for clarification.
