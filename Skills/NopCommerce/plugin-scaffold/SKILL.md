---
name: plugin-scaffold
description: >-
  Scaffold a brand-new nopCommerce plugin from scratch: folder structure,
  plugin.json, {Name}Defaults, BaseNameCompatibility, SchemaMigration,
  NopStartup DI registration, ViewLocationExpander, and RouteProvider,
  including NopStation-specific naming rules and per-plugin-type guidance
  (payment, widget, shipping, tax, etc.). Use when starting a new plugin
  project — always ask first whether it's a NopStation or custom plugin,
  then load the matching type-specific scaffold file once the plugin type
  is known.
---

# Plugin-Scaffold Skill

## When to Use

Activate this skill when:
- Always ask if this is a NopStation plugin or a custom plugin
- Creating a brand-new nopCommerce plugin from scratch
- Scaffolding the initial folder structure and boilerplate files
- Setting up the standard infrastructure (DI, view locations, migrations)

**Type-specific guidance**: This skill directory contains separate files for each plugin type (e.g., `payment-plugin.md`, `widget-plugin.md`).
Once the plugin type is identified, also load the corresponding type-specific scaffold file to get exact interface requirements,
class templates, and unique architecture patterns. See the Plugin Types table below for the mapping.

## nopCommerce Plugin Architecture
- Every plugin requires `.csproj` targeting .NET version (9.0/8.0/7.0) and `plugin.json` metadata file
- Plugin class must inherit from `BasePlugin`
- Override `InstallAsync()`, `UninstallAsync()`, `UpdateAsync()` for lifecycle hooks
- Always call `base.InstallAsync()` / `base.UninstallAsync()` when overriding

## nopStation Plugin Conventions
Agent must follow these conventions when scaffolding a plugin for nopStation. The namespace MUST start with `NopStation.Plugin.` — this is non-negotiable for NopStation plugins:
- Plugin namespace: `NopStation.Plugin.{Group}.{Name}` (e.g., `NopStation.Plugin.Widgets.FairPrice`)
- Plugin folder: `NopStation.Plugin.{Group}.{Name}/`
- Always use `NS_` as table prefix.

## How to Use

### Step 1 — Gather Requirements

Before generating, ask the user for:

1. **Plugin namespace** (e.g., `NopStation.Plugin.Widgets.FAQ`)
2. **Plugin type** — `Misc`, `Widgets`, `Payments`, `Shipping`, `Tax`, etc.
   - Once identified, load the corresponding `{type}-plugin.md` file from this directory for interface-specific guidance
3. **TablePrefix** — e.g., `NS_` (ALWAYS ask, never assume)
4. **Domain entities** — what tables/entities are needed
5. **Has admin area?** — almost always yes
6. **Has public area?** — depends on plugin type
7. **Has API endpoints?** — if so, also use the `API-Design` skill

### Plugin Types and Naming Conventions

```
NopStation.Plugin.{Group}.{Name}
```

| Group | Used For | Interface | Scaffold File |
|-------|---------|---------|--------------|
| `Payment` | Payment gateways (PayPal, Stripe, etc.) | IPaymentMethod | `payment-plugin.md` |
| `Shipping` | Shipping rate calculators | IShippingRateComputationMethod | `shipping-plugin.md` |
| `Tax` | Tax providers | ITaxProvider | `tax-provider-plugin.md` |
| `Widgets` | UI injections via widget zones | IWidgetPlugin | `widget-plugin.md` |
| `Misc` | General-purpose business plugins | IMiscPlugin | `misc-plugin.md` |
| `ExchangeRate` | Currency exchange providers | IExchangeRateProvider | `exchangerate-provider-plugin.md` |
| `DiscountRules` | Discount condition rules | IDiscountRequirementRule | `discountrules-plugin.md` |
| `Pickup` | Pickup point providers | IPickupPointProvider | `pickup-plugin.md` |
| `ExternalAuth` | External auth providers | IExternalAuthenticationMethod | `externalauth-plugin.md` |
| `SearchPluginManager` | Search plugin providers | IMiscPlugin | `search-provider-plugin.md` |
| `MultiFactorAuth` | Multi-factor authentication methods | IMultiFactorAuthenticationMethod | `multifactor-auth-plugin.md` |

Examples:
```
Nop.Plugin.Payment.PayPalStandard
Nop.Plugin.Widgets.GoogleAnalytics
Nop.Plugin.Misc.MyBusinessFeature
Nop.Plugin.Shipping.FixedByWeightByTotal
```

### Step 2 — Generate Folder Structure

```
NopStation.Plugin.{Group}.{Name}/
│
├── 📄 NopStation.Plugin.{Group}.{Name}.csproj  # Project file (.NET 9)
├── 📄 plugin.json                        # Plugin manifest (required)
├── 📄 {Name}Plugin.cs                    # Main plugin class
├── 📄 {Name}Defaults.cs                  # Constants and file paths
├── 📄 {Name}PermissionProvider.cs        # Custom permissions (Plugin specific)
├── 📄 AdminMenuCreatedEventConsumer.cs
├── 📄 release_note.txt
│
├── 📁 Areas/                             # MVC Areas (recommended)
│   └── 📁 Admin/                         # Admin area
│       ├── 📁 Controllers/               # Admin controllers
│       ├── 📁 Factories/                 # Model factories
|       ├── 📁 Infrastructure/            # Admin specific DI and startup
|       │   ├── 📁 Mapper/
|       │
│       ├── 📁 Models/                    # Admin view models
│       │   └── ConfigurationModel.cs
│       ├── 📁 Validators/                # FluentValidation
│       └── 📁 Views/                     # Admin Razor views
│           ├── _ViewImports.cshtml
│           └── _ViewStart.cshtml
│
├── 📁 Components/                        # ViewComponents
│   └── CustomViewComponent.cs            # Widget view components
│
├── 📁 Controllers/                       # Public controllers
│
├── 📁 Data/                              # Data access layer
│   ├── 📁 Builders/                      # Entity mapping builders
│   ├── 📁 Migrations/                    # Update migrations
│   ├── BaseNameCompatibility.cs          # Table name compatibility
│   └── SchemaMigration.cs                # Initial schema migration
│
├── 📁 Domains/                           # Domain entities
│   └── 📁 Enums/                         # Enumerations
│
├── 📁 Events/                            # Custom events
│
├── 📁 Extensions/                        # Extension methods
│
├── 📁 Factories/                         # Public model factories
│
├── 📁 Helpers/                           # Utility classes
│
├── 📁 Infrastructure/                    # DI, startup, and route registration
│   ├── 📁 Mapper/                        # AutoMapper profiles
│   ├── PluginNopStartup.cs               # Service registration
│   ├── RouteProvider.cs                  # Named route registration (IRouteProvider)
│
├── 📁 Models/                            # Public view models
│
├── 📁 Services/                          # Business logic
│   ├── 📁 Cache/                         # Cache key definitions
│   └── EventConsumer.cs                 # Event handlers
│
├── 📁 Settings/                          # Plugin settings classes
│
└── 📁 Views/                             # Public views
    ├── 📁 Shared/                        # Shared views
    │   └── 📁 Components/                # ViewComponent views
    │       └── 📁 Custom/                # Named after component
    │           └── Default.cshtml       # Default view
    ├── _ViewImports.cshtml
    └── _ViewStart.cshtml
```

### Step 3 — Generate Boilerplate Files

Generate files in this order (dependencies first):

1. **`plugin.json`** — Plugin manifest
2. **`{Name}Defaults.cs`** — Constants (SystemName, TablePrefix, paths)
3. **`Domains/`** — Entity classes
4. **`Data/BaseNameCompatibility.cs`** — Table name mapping
5. **`Data/Builders/`** — Entity builders
6. **`Data/SchemaMigration.cs`** — Initial schema with existence check
7. **`Settings/`** — Plugin settings classes
8. **`Services/`** — Service interfaces and implementations
9. **`Infrastructure/NopStartup.cs`** — DI registrations
10. **`Infrastructure/ViewLocationExpander.cs`** — View paths
11. **`PluginPermissionConfigManager.cs`** — Permissions
12. **`Plugin.cs`** — Main plugin class
13. **`Localization/pluginResources.en-us.xml`** — Resources
14. **Admin area** — Controllers, factories, models, validators, views
15. **Public area** — Controllers, factories, models, views

### Step 4 — Key Patterns

#### plugin.json — Complete Reference

```json
{
    "Group": "{Group}",
    "FriendlyName": "Plugin Display Name",
    "SystemName": "NopStation.Plugin.{Group}.{Name}",
    "Version": "4.90.1.1",
    "SupportedVersions": ["4.90"],
    "Author": "Your Company Name",
    "DisplayOrder": 1,
    "FileName": "NopStation.Plugin.{Group}.{Name}.dll",
    "Description": "One-line description of what this plugin does.",
    "LimitedToStores": [],
    "LimitedToCustomerRoles": [],
    "DependsOnSystemNames": []
}
```

Rules:
- `SystemName` must be globally unique — use the assembly name
- `Version` must include `SupportedVersions` as major.minor, patch version can be plugin-specific. e.g., `4.90.1.1`
- `SupportedVersions` must include `"4.90"` for this project
- `FileName` must match the compiled output DLL name exactly
- `DependsOnSystemNames` lists other plugin SystemNames this depends on
- `DisplayOrder` controls ordering in the plugin list (lower = higher)

#### {Name}Defaults.cs
```csharp
public static class {Name}Defaults
{
    public static string TablePrefix => "NS_";
    public static string XmlResourceStringFilePath => "~/Plugins/{PluginNamespace}/Localization/pluginResources.en-us.xml";
    public static string PluginSystemName => "{PluginNamespace}";
    public static string PluginMenuSystemName => "NopStation.{Name}";

    /// <summary>
    /// Route names (used with INopUrlHelper.RouteUrl() for URL generation)
    /// </summary>
    public static class Route
    {
        public static string Configuration => "Plugin.{PluginSystemName}.Configure";
    }
}
```

#### BaseNameCompatibility.cs
```csharp
public class BaseNameCompatibility : INameCompatibility
{
    public Dictionary<Type, string> TableNames => new()
    {
        { typeof(MyEntity), $"{PluginNameDefaults.TablePrefix}{nameof(MyEntity)}" }
    };

    public Dictionary<(Type, string), string> ColumnName => new() { };
}
```

#### SchemaMigration.cs — ALWAYS check table existence
```csharp
[NopSchemaMigration("2025/01/01 00:00:00", "NopStation.Plugin.{Group}.Name} schema migration", MigrationProcessType.Installation)]
public class SchemaMigration : AutoReversingMigration
{
    public override void Up()
    {
        Create.TableFor<MyEntity>();
    }
}
```

#### ViewLocationExpander.cs
```csharp
public class ViewLocationExpander : IViewLocationExpander
{
    protected const string THEME_KEY = "nop.themename";

    public IEnumerable<string> ExpandViewLocations(
        ViewLocationExpanderContext context, IEnumerable<string> viewLocations)
    {
        if (context.AreaName == "Admin")
        {
            viewLocations = new[]
            {
                $"/Plugins/{PluginNamespace}/Areas/Admin/Views/Shared/{{0}}.cshtml",
                $"/Plugins/{PluginNamespace}/Areas/Admin/Views/{{1}}/{{0}}.cshtml"
            }.Concat(viewLocations);
        }
        else
        {
            viewLocations = new[]
            {
                $"~/Plugins/{PluginNamespace}/Views/Shared/{{0}}.cshtml",
                $"~/Plugins/{PluginNamespace}/Views/{{1}}/{{0}}.cshtml"
            }.Concat(viewLocations);

            if (context.Values.TryGetValue(THEME_KEY, out string theme))
            {
                viewLocations = new[]
                {
                    $"/Plugins/{PluginNamespace}/Themes/{theme}/Views/Shared/{{0}}.cshtml",
                    $"/Plugins/{PluginNamespace}/Themes/{theme}/Views/{{1}}/{{0}}.cshtml"
                }.Concat(viewLocations);
            }
        }
        return viewLocations;
    }

    public void PopulateValues(ViewLocationExpanderContext context)
    {
        if (context.AreaName?.Equals(AreaNames.ADMIN) ?? false)
            return;
        context.Values[THEME_KEY] = EngineContext.Current
            .Resolve<IThemeContext>().GetWorkingThemeNameAsync().Result;
    }
}
```

#### Plugin.cs — GetConfigurationPageUrl, Install/Uninstall/Update
```csharp
public class PluginName : BasePlugin, IWidgetPlugin
{
    #region Fields

    private readonly INopUrlHelper _nopUrlHelper;
    private readonly ILocalizationService _localizationService;
    private readonly ISettingService _settingService;

    #endregion

    #region Ctor

    public PluginName(INopUrlHelper nopUrlHelper,
        ILocalizationService localizationService,
        ISettingService settingService)
    {
        _nopUrlHelper = nopUrlHelper;
        _localizationService = localizationService;
        _settingService = settingService;
    }

    #endregion

    #region Methods

    public override string GetConfigurationPageUrl()
    {
        return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
    }

    public override async Task InstallAsync()
    {
        await _settingService.SaveSettingAsync(new PluginSettings { EnablePlugin = true });
        await InstallLocalResourceStringFromXmlFileAsync();
        await InstallPermissionsAsync();
        await base.InstallAsync();
    }

    public override async Task UninstallAsync()
    {
        // Remove permissions (NOT user data)
        await base.UninstallAsync();
    }

    public override async Task UpdateAsync(string currentVersion, string targetVersion)
    {
        if (!currentVersion.Equals(targetVersion))
        {
            await InstallLocalResourceStringFromXmlFileAsync();
            await InstallPermissionsAsync();
        }
        await base.UpdateAsync(currentVersion, targetVersion);
    }

    #endregion
}
```

#### NopStartup.cs
```csharp
public class NopStartup : INopStartup
{
    public int Order => 100;

    public void Configure(IApplicationBuilder application) { }

    public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {
        services.Configure<RazorViewEngineOptions>(options =>
        {
            options.ViewLocationExpanders.Add(new ViewLocationExpander());
        });

        // Services
        services.AddScoped<IMyService, MyService>();

        // Admin Factories
        services.AddScoped<IConfigurationModelFactory, ConfigurationModelFactory>();

        // Public Factories
        services.AddScoped<IPublicModelFactory, PublicModelFactory>();
    }
}
```

#### RouteProvider.cs — Named Route Registration

Register named routes via `IRouteProvider` so that `INopUrlHelper.RouteUrl()` can resolve them:

```csharp
public class RouteProvider : BaseRouteProvider, IRouteProvider
{
    public void RegisterRoutes(IEndpointRouteBuilder endpointRouteBuilder)
    {
        endpointRouteBuilder.MapControllerRoute(name: {Name}Defaults.Route.Configuration,
            pattern: "Admin/{ControllerName}/Configure",
            defaults: new { controller = "{ControllerName}", action = "Configure", area = AreaNames.ADMIN });
    }

    public int Priority => 0;
}
```

Usage in Plugin.cs:
```csharp
public override string GetConfigurationPageUrl()
{
    return _nopUrlHelper.RouteUrl({Name}Defaults.Route.Configuration);
}
```

#### .csproj — Content Items (Use Glob Patterns)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <OutputPath>..\..\Presentation\Nop.Web\Plugins\NopStation.Plugin.{Group}.{Name}</OutputPath>
    <OutDir>$(OutputPath)</OutDir>
    <CopyLocalLockFileAssemblies>false</CopyLocalLockFileAssemblies>
  </PropertyGroup>

  <ItemGroup>
    <!-- Wildcard: catches all cshtml without per-file entries -->
    <Content Include="Areas\Admin\Views\**\*.cshtml">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <Content Include="Views\**\*.cshtml">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <!-- Only add Themes glob if plugin has theme views -->
    <Content Include="Themes\**\Views\**\*.cshtml">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <!-- Non-cshtml assets stay explicit -->
    <Content Include="logo.png">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
    <Content Include="plugin.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>

  <ItemGroup>
    <ClearPluginAssemblies Include="$(MSBuildProjectDirectory)\..\..\Build\ClearPluginAssemblies.proj" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\NopStation.Plugin.Misc.Core\NopStation.Plugin.Misc.Core.csproj">
      <Private>false</Private>
    </ProjectReference>
  </ItemGroup>

</Project>
```

**Rule:** Always use glob patterns for cshtml files — never list each cshtml individually.
Do not add `<None Remove="...">` entries for cshtml files; the glob `Content Include` handles them.


## Guardrails
- Never assume, always ask if this is a NopStation plugin or a custom plugin
- For NopStation plugins, namespace MUST start with `NopStation.Plugin.` — this convention is non-negotiable
- Use `NopStation` prefix only for NopStation plugins, otherwise ask the user for the prefix
