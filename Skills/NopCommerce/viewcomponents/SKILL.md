---
name: viewcomponents
description: >-
  Create ViewComponents (inheriting NopViewComponent) for nopCommerce plugins
  — widget-zone content, payment-info/checkout blocks, and other reusable UI
  fragments — covering the (widgetZone, additionalData) InvokeAsync
  signature, guard-clause ordering, view-resolution conventions, and
  IWidgetPlugin registration. Use when rendering a UI fragment from a
  controller or view, building a widget plugin's zone content, or adding
  payment/checkout UI.
---

# ViewComponents in nopCommerce

## When to Use

Activate this skill when:
- Creating a widget plugin that renders content in widget zones
- Adding payment info/input blocks to checkout
- Rendering reusable UI fragments from controllers or views
- Extending admin or public pages with additional panels

## Base Class

All ViewComponents inherit from `NopViewComponent` in `Nop.Web.Framework.Components`. This base extends ASP.NET Core's `ViewComponent` and fires `ModelPrepared` events automatically.

```csharp
using Microsoft.AspNetCore.Mvc;
using Nop.Web.Framework.Components;

namespace Nop.Plugin.Widgets.Example.Components;

public class ExampleViewComponent : NopViewComponent
{
}
```

## Folder Structure

ViewComponents live in a `Components/` folder under the plugin root. For plugins with many components, organize into sub-namespaces:

```
Plugin.Root/
├── Components/
│   ├── ExampleViewComponent.cs
│   ├── Public/
│   │   └── ButtonsViewComponent.cs
│   └── Admin/
│       └── PaymentMethodViewComponent.cs
└── Areas/Admin/Components/
    └── AdminOnlyViewComponent.cs
```

## InvokeAsync Method — Widget Signature

Widget ViewComponents use the standard `(string widgetZone, object additionalData)` signature:

```csharp
public async Task<IViewComponentResult> InvokeAsync(string widgetZone, object additionalData)
{
    if (additionalData is not ProductDetailsModel model)
        return Content(string.Empty);

    return View(model);
}
```

Non-widget ViewComponents can use any signature:

```csharp
public async Task<IViewComponentResult> InvokeAsync()
{
    var model = await _modelFactory.PrepareModelAsync();
    return View(model);
}
```

## Return Types

| Return | When to Use | Example |
|--------|-------------|---------|
| `Content(string.Empty)` | Early exit / guard clause | `return Content(string.Empty);` |
| `View(model)` | Render a named view by convention | `return View(model);` |
| `View("~/Plugins/.../Views/...cshtml", model)` | Explicit full path to view | `return View("~/Plugins/Payments.MyPlugin/Views/PaymentInfo.cshtml", model);` |
| `new HtmlContentViewComponentResult(...)` | Raw HTML without a view file | `return new HtmlContentViewComponentResult(new HtmlString(script));` |

## Guard Clauses

Always guard early — check widget zone, permissions, plugin activation, and data type before doing work:

```csharp
public async Task<IViewComponentResult> InvokeAsync(string widgetZone, object additionalData)
{
    if (!widgetZone.Equals(AdminWidgetZones.PaymentMethodListTop))
        return Content(string.Empty);

    if (!await _permissionService.AuthorizeAsync(StandardPermission.Configuration.MANAGE_TAX_SETTINGS))
        return Content(string.Empty);

    if (!await _taxPluginManager.IsPluginActiveAsync(AvalaraTaxDefaults.SystemName))
        return Content(string.Empty);

    if (additionalData is not BaseNopEntityModel entityModel)
        return Content(string.Empty);

    // ... main logic
}
```

## Widget Registration

For widget plugins, register the ViewComponent via `IWidgetPlugin`:

```csharp
public class MyWidgetPlugin : BasePlugin, IWidgetPlugin
{
    public bool HideInWidgetList => false;

    public Type GetWidgetViewComponent(string widgetZone)
    {
        return typeof(ExampleViewComponent);
    }

    public Task<IList<string>> GetWidgetZonesAsync()
    {
        return Task.FromResult<IList<string>>(new List<string>
        {
            PublicWidgetZones.ProductDetailsTop
        });
    }
}
```

## Widget Zone Constants

Always use the constants — never hardcode zone strings:

**Public zones** — `Nop.Web.Framework.Infrastructure.PublicWidgetZones`
**Admin zones** — `Nop.Web.Framework.Infrastructure.AdminWidgetZones`

## Invoking from Views

Use `@await Component.InvokeAsync(typeof(YourViewComponent))` from any Razor view:

```html
@await Component.InvokeAsync(typeof(StoreScopeConfigurationViewComponent))
```

For widget ViewComponents, nopCommerce's widget engine calls them automatically based on the `IWidgetPlugin` configuration.

## View Naming Conventions

When returning `View(model)` without an explicit path:
- ASP.NET Core looks for `Views/Components/{ComponentName}/{ViewName}.cshtml`
- Default view name is `Default.cshtml`
- Component name strips the `ViewComponent` suffix from the class name

Example: `ExampleViewComponent` with `return View(model)` looks for `Views/Components/Example/Default.cshtml`.

When using explicit paths, place views anywhere:

```csharp
return View("~/Plugins/Payments.MyPlugin/Views/PaymentInfo.cshtml", model);
```

## Templates

### Basic Widget ViewComponent

```csharp
using Microsoft.AspNetCore.Mvc;
using Nop.Web.Framework.Components;

namespace Nop.Plugin.Widgets.Example.Components;

public class ExampleViewComponent : NopViewComponent
{
    public async Task<IViewComponentResult> InvokeAsync(string widgetZone, object additionalData)
    {
        return Content(string.Empty);
    }
}
```

### ViewComponent with View

```csharp
using Microsoft.AspNetCore.Mvc;
using Nop.Web.Framework.Components;

namespace Nop.Plugin.Misc.Example.Components;

public class InfoViewComponent : NopViewComponent
{
    public async Task<IViewComponentResult> InvokeAsync()
    {
        var model = new InfoModel { Message = "Hello from ViewComponent" };
        return View(model);
    }
}
```

### Admin Area ViewComponent (non-NopStation)

For admin-only components under `Areas/Admin/Components/`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Nop.Web.Framework.Components;

namespace Nop.Plugin.Misc.Example.Areas.Admin.Components;

public class AdminPanelViewComponent : NopViewComponent
{
    public async Task<IViewComponentResult> InvokeAsync(string widgetZone, object additionalData)
    {
        if (!widgetZone.Equals(AdminWidgetZones.ProductDetailsBlock))
            return Content(string.Empty);

        return View(model);
    }
}
```

## Guardrails

- **Always inherit from `NopViewComponent`** — not from `ViewComponent` directly
- **Always use widget zone constants** — never hardcode zone strings
- **Use `Content(string.Empty)` for early exits** — never return `null`
- **Make `InvokeAsync` return `Task<IViewComponentResult>`** — avoid synchronous `Invoke`
- **Guard at the top** — check zones, permissions, and data types before any logic
- **Prefer explicit view paths** — `View("~/Plugins/...")` avoids ambiguity
- **Do not inject business logic** — delegate to services; ViewComponents should orchestrate, not implement data access
- **Do not use `ILogger<T>`** — use nopCommerce's `ILogger` if logging is needed
- **Do not hardcode strings** — use localization for any user-facing text
