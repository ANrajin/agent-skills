---
name: controllers
description: >-
  Create MVC controllers for nopCommerce plugins, picking the correct base
  class (BaseAdminController/BasePublicController for custom plugins,
  NopStationAdminController/NopStationPublicController for NopStation
  plugins) and avoiding attributes, try-catch blocks, or manual validation
  the base class already provides. Use when adding a new admin or public
  controller, or a new action method, to a plugin.
---

# Controllers in nopCommerce

## When to use
Activate this skill when you need to create controllers in nopCommerce plugin.

## How to use
- Identity whether it is a NopStation plugin or custom plugin.
**For Custom Plugin:**
- Inherit the `BaseAdminController` only when the controller is for the admin area. Otherwise, inherit from `BasePublicController`.
**For NopStation Plugin:**
- Inherit the `NopStationAdminController` only when the controller is for the admin area. Otherwise, inherit from `NopStationPublicController`.

## Templates
```C#
namespace Nop.Plugin.{Group}.{Name}.Areas.Admin.Controllers

public class YourController : BaseAdminController
{
}
```

```C#
namespace Nop.Plugin.{Group}.{Name}.Controllers

public class YourController : BasePublicController
{
}
```

## Guardrails
- When defining controllers that inherit from `BaseAdminController` or `NopStationAdminController`, do not define the following attributes, as they are already inherited:
    - `[Area(AreaNames.ADMIN)]`
    - `[AutoValidateAntiforgeryToken]`
    - `[ValidateIpAddress]`
    - `[AuthorizeAdmin]`
    - `[ValidateVendor]`
    - `[SaveSelectedTab]`
    - `[NotNullValidationMessage]`
- **Exception:** Discount rules plugins (`IDiscountRequirementRule`) are a special case — their admin controller inherits `BasePluginController` instead of `BaseAdminController`.
- Do not explicitly define `try-catch` blocks in controllers, as nopCommerce automatically handles error logging
- Never use `ILogger<T>` from `Microsoft.Extensions.Logging`
- Always use `FluentValidation` for model validation; refrain from writing manual validation in controllers.
