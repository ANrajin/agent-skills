---
name: fluentvalidator
description: >
  Use this skill whenever creating or editing FluentValidation validator classes
  in a nopCommerce plugin. Triggers include: adding validation to a request/model
  class, writing a new *Validator.cs file, adding RuleFor() chains, integrating
  localization into validation messages, or wiring up BaseNopValidator. Use even
  if user just says "add validation to my model" or "validate this request" inside
  a nopCommerce plugin context.
---

# nopCommerce FluentValidation Skill

## When to Use

Use this skill when:
- Creating a new `*Validator.cs` file for any plugin model or request class
- Adding or editing `RuleFor()` validation rules on an existing validator
- Wiring localization messages into validation errors
- Validating request DTOs received by plugin API/controller actions
- User says "validate this model", "add validation", "check input" in nopCommerce plugin context

Do NOT use this skill for:
- Validation in non-plugin nopCommerce core code (different base classes apply)
- Client-side / JavaScript validation
- Data annotation attributes (`[Required]`, `[MaxLength]`) — those are a separate pattern

---

## How to Use

### Step 1: Identify validator type

Two validator types exist. Pick based on where the model lives:

| Type | Model location | Rules placement |
|------|---------------|-----------------|
| **API / Request validator** | `YourPlugin/Validators/` | Inside `RuleSet(NopValidationDefaults.ValidationRuleSet, () => { })` |
| **Admin model validator** | `YourPlugin/Areas/Admin/Validators/` | Directly in constructor — NO `RuleSet` wrapper |

### Step 2: Create the validator file

**API / Request validator** — `YourPlugin/Validators/{ModelName}Validator.cs`

```csharp
using FluentValidation;
using Nop.Services.Localization;
using Nop.Web.Framework.Validators;
using YourPlugin.Models;

namespace YourPlugin.Validators;

public partial class {ModelName}Validator : BaseNopValidator<{ModelName}>
{
    public {ModelName}Validator(ILocalizationService localizationService)
    {
        RuleSet(NopValidationDefaults.ValidationRuleSet, () =>
        {
            RuleFor(x => x.FieldName)
                .[ValidationMethod]()
                .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.Field.ErrorType"));
        });
    }
}
```

**Admin model validator** — `YourPlugin/Areas/Admin/Validators/{ModelName}Validator.cs`

```csharp
using FluentValidation;
using Nop.Services.Localization;
using Nop.Web.Framework.Validators;
using YourPlugin.Areas.Admin.Models;

namespace YourPlugin.Areas.Admin.Validators;

public partial class {ModelName}Validator : BaseNopValidator<{ModelName}>
{
    public {ModelName}Validator(ILocalizationService localizationService)
    {
        RuleFor(x => x.FieldName)
            .[ValidationMethod]()
            .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.Field.ErrorType"));
    }
}
```

### Step 3: Register locale keys

All resource strings live in `GetPluginResources()` inside the `{PluginName}Plugin.cs` file.
```C#
["Plugin.{PluginName}.{ModelName}.Name.Required"] = "Name is required."
```
**Or**
Add to `locales/en-US.xml`, if xml based local keys are configured:
```xml
<LocaleResource Name="Plugin.YourPlugin.Model.Field.Required">
  <Value>Field is required.</Value>
</LocaleResource>
```

Locale key pattern: `Plugin.{PluginSystemName}.{ModelName}.{FieldName}.{ErrorType}`

### Step 4: File structure

```
YourPlugin/
├── Validators/
│   └── {ModelName}Validator.cs          <- API/request validators
├── Models/
│   └── {ModelName}.cs
├── Areas/
│   └── Admin/
│       ├── Validators/
│       │   └── {ModelName}Validator.cs  <- admin model validators
│       └── Models/
│           └── {ModelName}Model.cs
```

**Register in NopStartup**

```csharp
#region Validators

services.AddScoped<IValidator<{ModelName}>, {ModelName}Validator>();

#endregion
```

---

## Common Patterns

### Required Guid
```csharp
RuleFor(x => x.SessionId)
    .NotEqual(Guid.Empty)
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.SessionId.Required"));
```

### Enum must be valid value
```csharp
RuleFor(x => x.SyncType)
    .IsInEnum()
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.SyncType.Invalid"));
```

### Positive integer
```csharp
RuleFor(x => x.TotalItems)
    .GreaterThan(0)
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.TotalItems.Positive"));
```

### Required string
```csharp
RuleFor(x => x.Name)
    .NotEmpty()
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.Name.Required"));
```

### String with max length
```csharp
RuleFor(x => x.Name)
    .NotEmpty()
    .MaximumLength(400)
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.Name.Required"));
```

### Valid email
```csharp
RuleFor(x => x.Email)
    .NotEmpty()
    .EmailAddress()
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.Email.InvalidEmail"));
```

### Conditional rule
```csharp
RuleFor(x => x.ExpiryDate)
    .GreaterThan(DateTime.UtcNow)
    .When(x => x.ExpiryDate.HasValue)
    .WithMessageAwait(localizationService.GetResourceAsync("Plugin.YourPlugin.Model.ExpiryDate.MustBeFuture"));
```

---

## Guardrails
- NEVER inherit `AbstractValidator<T>` directly — always use `BaseNopValidator<T>`
- NEVER use `.WithMessage("hardcoded string")` — always use `.WithMessageAwait(localizationService.GetResourceAsync(...))`
- NEVER use `RuleSet` in admin model validators (Areas/Admin/Validators/) — rules go directly in constructor
- NEVER omit `RuleSet(NopValidationDefaults.ValidationRuleSet, ...)` in API/request validators (Validators/) — nopCommerce won't execute bare rules
- FluentValidation constructors are synchronous. Async validators in the ASP.NET automatic validation pipeline cause issues. This is the established nopCommerce pattern.
- NEVER use `.Result` or `.GetAwaiter().GetResult()` on `GetResourceAsync` — `.WithMessageAwait()` handles async correctly
- ALWAYS make validator class `partial`
- ALWAYS add locale keys to the XML file for every new `.GetResourceAsync(...)` call — missing keys return empty string silently
