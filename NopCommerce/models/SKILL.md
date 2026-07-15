---
name: models
description: Create and maintain view models for nopCommerce plugins following established patterns for API request/response DTOs, admin configuration models, Kendo grid search/list models, entity display models, and AJAX response models.
---

# nopCommerce Plugin View Models Skill

## When to Use

Activate this skill when:
- Adding new view model. Public or admin area.
- Adding a new API endpoint to a plugin controller that requires request/response models.
- Creating new admin Kendo grid list pages with search, list, and entity models.
- Adding settings or configuration fields to an existing plugin admin configuration model.
- Building an AJAX action in an admin or public controller that returns a JSON response model.
- Adding a new admin card or settings section to a plugin configuration page.
- Extending admin UI lists with new filters, search blocks, or columns.
- Introducing a new domain entity in a plugin that needs an admin display model.

---

## How to Use

### Step 1 — Identify the Model Category

When creating a new model, identify which category it belongs to and use the corresponding base class, namespace, and naming conventions:

| Category | Namespace | Base Class / Interface | Purpose |
|----------|-----------|------------------------|---------|
| **API Request DTO** | `Nop.Plugin.{Group}.{Name}.Models` | `record` or class (no base) | Inbound JSON payload from external APIs |
| **API Response DTO** | `Nop.Plugin.{Group}.{Name}.Models` | `record` or class (no base) | Outbound JSON payload to external APIs |
| **API Envelope** | `Nop.Plugin.{Group}.{Name}.Models` | `record` or class (no base) | Generic response wrappers (e.g., standard success/error structures) |
| **Admin Configuration** | `Nop.Plugin.{Group}.{Name}.Areas.Admin.Models` or `.Models` | `BaseNopModel`, `ISettingsModel` | Plugin configuration page settings |
| **Admin Search** | `Nop.Plugin.{Group}.{Name}.Areas.Admin.Models` or `.Models` | `BaseSearchModel` | Search panel filter bindings for Kendo grid |
| **Admin List** | `Nop.Plugin.{Group}.{Name}.Areas.Admin.Models` or `.Models` | `BasePagedListModel<TEntityModel>` | Wrapper for paged grid rows |
| **Admin Entity Display** | `Nop.Plugin.{Group}.{Name}.Areas.Admin.Models` or `.Models` | `BaseNopEntityModel` or `BaseNopModel` | Representation of a single row in Kendo grid |
| **AJAX/JSON Response** | `Nop.Plugin.{Group}.{Name}.Areas.Admin.Models` or `.Models` | `BaseNopModel` or plain class/record | Responses from AJAX action methods |

---

### Step 2 — Implement API Request/Response DTOs

For external API integrations, DTOs should reside in the root `Models/` folder of the plugin. They should be lightweight, immutable (using C# `record` with `init` accessors), and explicitly decorated for serialization.

#### Request DTO Example

```csharp
using Newtonsoft.Json;

namespace Nop.Plugin.Misc.MyPlugin.Models;

public record SessionStartRequest
{
    [JsonProperty("sessionId")]
    public Guid SessionId { get; init; }

    [JsonProperty("totalItems")]
    public int TotalItems { get; init; }

    [JsonProperty("syncType")]
    public string SyncType { get; init; }
}
```

#### Response DTO Example

```csharp
using Newtonsoft.Json;

namespace Nop.Plugin.Misc.MyPlugin.Models;

public record SessionStartResponse
{
    [JsonProperty("sessionId")]
    public Guid SessionId { get; init; }

    [JsonProperty("status")]
    public string Status { get; init; }

    [JsonProperty("createdAt")]
    public DateTime CreatedAt { get; init; }
}
```

---

### Step 3 — Implement Admin Configuration Model

The configuration model represents settings that can be customized by store administrators. If settings are overridable per store (multi-store configuration), the model must implement `ISettingsModel` and contain override properties.

```csharp
using Nop.Web.Framework.Models;
using Nop.Web.Framework.Mvc.ModelBinding;

namespace Nop.Plugin.Misc.MyPlugin.Areas.Admin.Models;

public record ConfigurationModel : BaseNopModel, ISettingsModel
{
    #region Properties

    public int ActiveStoreScopeConfiguration { get; set; }

    [NopResourceDisplayName("Plugins.MyPlugin.Fields.Enabled")]
    public bool Enabled { get; set; }
    public bool Enabled_OverrideForStore { get; set; }

    [NopResourceDisplayName("Plugins.MyPlugin.Fields.BatchSize")]
    public int BatchSize { get; set; }
    public bool BatchSize_OverrideForStore { get; set; }

    #endregion
}
```

**Key Rules:**
- Include `ActiveStoreScopeConfiguration` to track which store is being configured.
- For each property `PropertyName` that represents a setting, add a boolean property named `PropertyName_OverrideForStore` to handle store-specific configuration overrides.
- Decorate user-facing properties with `[NopResourceDisplayName("Resource.String.Key")]`.

---

### Step 4 — Implement Search and List Models for Kendo Grid

Search models bind filters from the admin grid UI. List models wrap the paged results bound to Kendo.

#### Search Model Example

```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Mvc.Rendering;
using Nop.Web.Framework.Models;
using Nop.Web.Framework.Mvc.ModelBinding;

namespace Nop.Plugin.Misc.MyPlugin.Areas.Admin.Models;

public record EntitySearchModel : BaseSearchModel
{
    #region Ctor

    public EntitySearchModel()
    {
        AvailableStatuses = new List<SelectListItem>();
    }

    #endregion

    #region Properties

    [NopResourceDisplayName("Plugins.MyPlugin.Fields.SearchName")]
    public string SearchName { get; set; }

    [NopResourceDisplayName("Plugins.MyPlugin.Fields.SearchStatus")]
    public int SearchStatusId { get; set; }

    public IList<SelectListItem> AvailableStatuses { get; set; }

    #endregion
}
```

#### List Model Example

```csharp
using Nop.Web.Framework.Models;

namespace Nop.Plugin.Misc.MyPlugin.Areas.Admin.Models;

public record EntityListModel : BasePagedListModel<EntityModel>
{
}
```

---

### Step 5 — Implement Entity Display Model

This model maps domain entity properties to localized and formatted string values suitable for display in Kendo Grid columns.

```csharp
using Nop.Web.Framework.Models;
using Nop.Web.Framework.Mvc.ModelBinding;

namespace Nop.Plugin.Misc.MyPlugin.Areas.Admin.Models;

public record EntityModel : BaseNopEntityModel
{
    [NopResourceDisplayName("Plugins.MyPlugin.Fields.Name")]
    public string Name { get; set; }

    [NopResourceDisplayName("Plugins.MyPlugin.Fields.Status")]
    public string Status { get; set; }

    public int StatusId { get; set; }

    [NopResourceDisplayName("Plugins.MyPlugin.Fields.CreatedOn")]
    public string CreatedOn { get; set; }
}
```

**Key Rules:**
- Inherit from `BaseNopEntityModel` (which automatically provides an `int Id` property).
- Convert dates to formatted `string` properties. Do not expose raw UTC `DateTime` properties to the grid.
- Provide a `string` property for localized enum names and a companion `int` (e.g. `StatusId`) for raw values if conditional styling is needed in views.

---

## Common Patterns

### Pattern 1 — Preparing Models in Model Factories

Controllers must not query databases directly or build models. Delegate model preparation to a dedicated Model Factory.

```csharp
public async Task<EntitySearchModel> PrepareEntitySearchModelAsync()
{
    var model = new EntitySearchModel();

    // Populate dropdown options
    model.AvailableStatuses.Add(new SelectListItem
    {
        Value = "0",
        Text = await _localizationService.GetResourceAsync("Admin.Common.All")
    });

    // Populate grid page size dropdown based on user settings
    model.SetGridPageSize();

    return model;
}
```

### Pattern 2 — Mapping Domain Paged Lists to List Models

Use the `.PrepareToGridAsync` extension to transform a paged list of domain entities into the paged list model for the grid.

```csharp
public async Task<EntityListModel> PrepareEntityListModelAsync(EntitySearchModel searchModel)
{
    ArgumentNullException.ThrowIfNull(searchModel);

    // Fetch paged data from service layer (0-indexed pageIndex)
    var entities = await _myService.GetPagedEntitiesAsync(
        name: searchModel.SearchName,
        statusId: searchModel.SearchStatusId,
        pageIndex: searchModel.Page - 1,
        pageSize: searchModel.PageSize);

    // Prepare list model using extension method
    var model = await new EntityListModel().PrepareToGridAsync(searchModel, entities, () =>
    {
        return entities.ToAsyncEnumerable().SelectAwait(async entity =>
        {
            return new EntityModel
            {
                Id = entity.Id,
                Name = entity.Name,
                StatusId = entity.StatusId,
                Status = await _localizationService.GetLocalizedEnumAsync((MyStatusEnum)entity.StatusId),
                CreatedOn = (await _dateTimeHelper.ConvertToUserTimeAsync(entity.CreatedOnUtc, DateTimeKind.Utc)).ToString("G")
            };
        });
    });

    return model;
}
```
---

## Guardrails

- **Separation of Concerns:** Keep models and factories thin. Do not place business logic, direct repository queries, or database transaction calls inside view models or model factories.
- **Null Safety:** Initialize all collections in model constructors (e.g., `AvailableStatuses = new List<SelectListItem>();`).
- **Use Localized Resources:** Never hardcode UI display strings. Always decorate model properties with `[NopResourceDisplayName("Key")]`.
- **String Defaults:** Prefer `string.Empty` over `null` for display string properties to avoid binding issues or null reference exceptions in razor views.
- **DateTime Handling:** Never expose raw `DateTime` properties to Admin list grids. Format them using `IDateTimeHelper` first.
- **Pascal Casing & JSON Serialization:** For API Request/Response DTOs, always match the serializer attributes configuration used in the project (usually `[JsonProperty("name")]` from `Newtonsoft.Json`).
- **Base Classes:** Ensure that admin view models extend the proper nopCommerce base classes:
  - Configuration: `BaseNopModel`
  - Entity Display: `BaseNopEntityModel`
  - Search: `BaseSearchModel`
  - List: `BasePagedListModel<T>`
- **Validation:** Do not write inline validation rules or attributes in the model itself. Always separate validation rules into a FluentValidation class inheriting `BaseNopValidator<TModel>`.
