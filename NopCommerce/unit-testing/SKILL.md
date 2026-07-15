---
name: unit-testing
description: >
  Write unit and integration tests for nopCommerce plugins following the core
  Nop.Tests patterns. Triggers include: creating a new plugin test project,
  adding tests for a plugin service/controller/validator/factory, setting up
  plugin-specific test infrastructure, or user asking to "write tests" or
  "add test coverage" for a plugin.
---

# nopCommerce Plugin Testing Skill

## When to Use

Activate this skill when:
- Creating a new test project for a plugin
- Adding test coverage for a plugin service, controller, validator, or model factory
- User says "write tests", "add unit tests", or "add test coverage" in a plugin context
- Setting up test infrastructure (base classes, test helpers, test plugins) for a plugin

Do NOT use this skill for:
- Testing non-plugin nopCommerce core code
- Integration or E2E tests outside the NUnit framework
- Performance / load testing

---

## How to Use

### Step 1 — Determine Test Scope

Identify what needs testing and choose the appropriate base class:

| Test Target | Base Class | Style | Mocking |
|-------------|-----------|-------|---------|
| **Service** | `ServiceTest` | Integration (real DB, real services) | Only ASP.NET infra mocked |
| **Service + CRUD** | `ServiceTest<TEntity>` | Integration (adds `CrudData<TEntity>` + auto `TestCrudAsync()`) | Same as ServiceTest |
| **Validator** | `BaseNopTest` | Unit (construct validator manually) | Dependencies resolved via `GetService<T>()` |
| **Model Factory** | `BaseNopTest` | Integration | Real DB, real services |
| **Controller** | `BaseNopTest` | Integration | Real DB, real services |

### Step 2 — Create the Test Project

Place the test project under `src/Tests/` following core convention:

```
src/Tests/
└── {PluginSystemName}.Tests/
    ├── {PluginSystemName}.Tests.csproj
    └── AssemblyInfo.cs
```

**csproj template:**

The `<TargetFramework>` element is intentionally omitted — `Directory.Build.props` at the repository root sets `TargetFramework=net9.0` and `ImplicitUsings=enable` globally, so all test projects inherit these settings.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <Description>Test project for {PluginFriendlyName}</Description>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="FluentAssertions" Version="7.2.0" />
    <PackageReference Include="Microsoft.Data.Sqlite" Version="9.0.9" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
    <PackageReference Include="Moq" Version="4.20.72" />
    <PackageReference Include="NUnit" Version="4.4.0" />
    <PackageReference Include="NUnit3TestAdapter" Version="5.1.0">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\Nop.Tests\Nop.Tests.csproj" />
    <ProjectReference Include="..\..\Plugins\{Group}\{PluginName}\{PluginName}.csproj" />
  </ItemGroup>

</Project>
```

Referencing the plugin project ensures its `INopStartup` implementation is discovered via assembly scanning when the test container is built, so plugin-specific services are automatically registered in the test DI container.

**AssemblyInfo.cs:**

```csharp
using NUnit.Framework;

[assembly: LevelOfParallelism(1)]
```

### Step 3 — Choose the Test Base Class

All test base classes derive from `Nop.Tests.BaseNopTest` which provides:
- A fully-built DI container with SQLite in-memory database
- All real nopCommerce services and repositories resolved from the container
- Sample data installed via `IInstallationService`
- `GetService<T>()` — two overloads:
  - `protected static T GetService<T>()` — resolves from the root container (most common)
  - `protected static T GetService<T>(IServiceScope scope)` — resolves from a scoped provider

```
BaseNopTest (Nop.Tests)
  ├── ServiceTest (Nop.Tests.Nop.Services.Tests)
  │     └── ServiceTest<TEntity> (Nop.Tests.Nop.Services.Tests)
  └── WebTest (Nop.Tests.Nop.Web.Tests)
```

**When plugin-specific test infrastructure is needed** (e.g., registering a fake plugin descriptor, overriding settings), create a plugin-specific base class in the test project.

When extending `ServiceTest`, the parent constructor already registers core test plugins in `Singleton<IPluginsInfo>.Instance`. Your constructor runs **after** the base, so add to the existing list:

```csharp
using Nop.Core.Infrastructure;
using Nop.Services.Plugins;
using Nop.Tests.Nop.Services.Tests;
using NUnit.Framework;

namespace Nop.Tests.Plugins.{Group}.{PluginName};

[TestFixture]
public abstract class {PluginName}ServiceTest : ServiceTest
{
    protected {PluginName}ServiceTest()
    {
        InitPlugin();
    }

    private static void InitPlugin()
    {
        Singleton<IPluginsInfo>.Instance.PluginDescriptors.Add((new PluginDescriptor
        {
            PluginType = typeof(YourPluginClass),
            SystemName = "Your.Plugin.SystemName",
            FriendlyName = "Your Plugin",
            Installed = true,
            ReferencedAssembly = typeof(YourPluginClass).Assembly
        }, true));
    }
}
```

If extending `BaseNopTest` directly (not `ServiceTest`), you must create a new `PluginsInfo` instance including both core test plugins and your own — follow the pattern in `ServiceTest.InitPlugins()`.

### Step 4 — Write Service Tests

Service tests use **real implementations** with a **real SQLite database**. Services and repositories are never mocked.

**Pattern:**

```csharp
using FluentAssertions;
using Nop.Tests;
using Nop.Tests.Nop.Services.Tests;
using NUnit.Framework;

namespace Nop.Tests.Plugins.{Group}.{PluginName}.Services;

[TestFixture]
public class {PluginService}Tests : {PluginName}ServiceTest
{
    private I{PluginService} _{pluginService};

    [OneTimeSetUp]
    public async Task SetUp()
    {
        _{pluginService} = GetService<I{PluginService}>();
        // seed test data here if needed
    }

    [OneTimeTearDown]
    public async Task TearDown()
    {
        // clean up test data
    }

    [Test]
    public async Task Can{MethodName}()
    {
        var result = await _{pluginService}.DoSomethingAsync();

        result.Should().NotBeNull();
    }
}
```

**Key rules:**
- Use `[OneTimeSetUp]` for fixture-level setup (run once)
- Use `[SetUp]` / `[TearDown]` for per-test setup/cleanup
- Resolve real services via `GetService<T>()`
- All assertions use **FluentAssertions**: `.Should().BeTrue()`, `.Should().NotBeNull()`, `.Should().Be(value)`, `.Should().BeGreaterThan(0)`

### Step 5 — Write CRUD Tests for Domain Entities

For entities that support insert/update/get/delete, extend `ServiceTest<TEntity>`. If you also need plugin-specific initialization, create a generic variant of your plugin base class:

```csharp
using FluentAssertions;
using Nop.Tests.Nop.Services.Tests;
using NUnit.Framework;

namespace Nop.Tests.Plugins.{Group}.{PluginName}.Services;

// Plugin-specific generic CRUD base (if plugin init is needed)
[TestFixture]
public abstract class {PluginName}ServiceTest<TEntity> : {PluginName}ServiceTest where TEntity : BaseEntity
{
    protected abstract CrudData<TEntity> CrudData { get; }

    [Test]
    public async Task TestCrudAsync()
    {
        var data = CrudData;
        data.BaseEntity.Id = 0;
        await data.Insert(data.BaseEntity);
        data.BaseEntity.Id.Should().BeGreaterThan(0);
        data.UpdatedEntity.Id = data.BaseEntity.Id;
        await data.Update(data.UpdatedEntity);
        var item = await data.GetById(data.BaseEntity.Id);
        item.Should().NotBeNull();
        data.IsEqual(data.UpdatedEntity, item).Should().BeTrue();
        await data.Delete(data.BaseEntity);
        item = await data.GetById(data.BaseEntity.Id);
        if (data.BaseEntity is ISoftDeletedEntity softDeletedEntity)
            softDeletedEntity.Deleted.Should().BeTrue();
        else
            item.Should().BeNull();
    }
}

// Concrete test class
[TestFixture]
public class {Entity}ServiceTests : {PluginName}ServiceTest<{Entity}>
{
    private I{Entity}Service _{entity}Service;

    [OneTimeSetUp]
    public async Task SetUp()
    {
        _{entity}Service = GetService<I{Entity}Service>();
    }

    protected override CrudData<{Entity}> CrudData
    {
        get
        {
            var baseEntity = new {Entity}
            {
                Name = "Test",
                Active = true
            };

            var updatedEntity = new {Entity}
            {
                Name = "Updated",
                Active = true
            };

            return new CrudData<{Entity}>
            {
                BaseEntity = baseEntity,
                UpdatedEntity = updatedEntity,
                Insert = _{entity}Service.InsertEntityAsync,
                Update = _{entity}Service.UpdateEntityAsync,
                Delete = _{entity}Service.DeleteEntityAsync,
                GetById = _{entity}Service.GetEntityByIdAsync,
                IsEqual = (first, second) => first.Name == second.Name && first.Active == second.Active
            };
        }
    }
}
```

If no plugin-specific init is needed, simply extend `ServiceTest<TEntity>` directly — see `AffiliateServiceTests` in the core test project for the pattern.

### Step 6 — Write Validator Tests

Validator tests instantiate the validator directly and use FluentValidation's `TestValidate()` extension:

```csharp
using FluentAssertions;
using FluentValidation.TestHelper;
using Nop.Services.Localization;
using Nop.Tests;
using NUnit.Framework;

namespace Nop.Tests.Plugins.{Group}.{PluginName}.Validators;

[TestFixture]
public class {ModelName}ValidatorTests : BaseNopTest
{
    private {ModelName}Validator _validator;

    [OneTimeSetUp]
    public void SetUp()
    {
        var localizationService = GetService<ILocalizationService>();
        _validator = new {ModelName}Validator(localizationService);
    }

    [Test]
    public void ShouldHaveErrorWhen{Field}IsInvalid()
    {
        var model = new {ModelName} { FieldName = string.Empty };
        _validator.TestValidate(model).ShouldHaveValidationErrorFor(x => x.FieldName);
    }

    [Test]
    public void ShouldNotHaveErrorWhen{Field}IsValid()
    {
        var model = new {ModelName} { FieldName = "valid value" };
        _validator.TestValidate(model).ShouldNotHaveValidationErrorFor(x => x.FieldName);
    }
}
```

**Test method naming for validators:**
- `ShouldHaveErrorWhen{Scenario}`
- `ShouldNotHaveErrorWhen{Scenario}`

**Assertion patterns:**
- `validator.TestValidate(model).ShouldHaveValidationErrorFor(x => x.Property)`
- `validator.TestValidate(model).ShouldNotHaveValidationErrorFor(x => x.Property)`
- `result.IsValid.Should().BeFalse()`

### Step 7 — Write Model Factory Tests

```csharp
using FluentAssertions;
using Nop.Tests;
using NUnit.Framework;

namespace Nop.Tests.Plugins.{Group}.{PluginName}.Factories;

[TestFixture]
public class {ModelName}ModelFactoryTests : BaseNopTest
{
    private I{ModelName}ModelFactory _factory;

    [OneTimeSetUp]
    public async Task SetUp()
    {
        _factory = GetService<I{ModelName}ModelFactory>();
    }

    [Test]
    public async Task Prepare{ModelName}ModelShouldPopulatePropertiesFromEntity()
    {
        var model = new {ModelName}Model();
        await _factory.Prepare{ModelName}ModelAsync(model, entity);

        model.Id.Should().Be(entity.Id);
        model.Name.Should().Be(entity.Name);
        // assert all mapped properties
    }
}
```

---

## Code Structure

### File Organization

```
src/Tests/{PluginName}.Tests/
├── {PluginName}.Tests.csproj
├── AssemblyInfo.cs
├── {PluginName}ServiceTest.cs           <- plugin-specific base class (if needed)
├── Services/
│   └── {PluginService}ServiceTests.cs
├── Validators/
│   └── {ModelName}ValidatorTests.cs
├── Factories/
│   └── {ModelName}ModelFactoryTests.cs
└── Controllers/
    └── {ControllerName}ControllerTests.cs
```

### Test Class Template

```csharp
using FluentAssertions;
using Nop.Tests;
using NUnit.Framework;

namespace Nop.Tests.Plugins.{Group}.{PluginName}.{Layer};

[TestFixture]
public class {Subject}Tests : {BaseClass}
{
    #region Fields

    private I{Service} _{service};

    #endregion

    #region Setup

    [OneTimeSetUp]
    public async Task SetUp()
    {
        _{service} = GetService<I{Service}>();
    }

    [OneTimeTearDown]
    public async Task TearDown()
    {
    }

    #endregion

    #region Tests

    [Test]
    public async Task Can{Action}()
    {
    }

    #endregion
}
```

---

## Naming Conventions

| Concept | Convention | Example |
|---------|-----------|---------|
| Test project | `{PluginSystemName}.Tests` | `Nop.Plugin.Misc.MyPlugin.Tests` |
| Test class | `{Subject}Tests` | `ProductSyncServiceTests` |
| Test method (service) | `Can{Action}` | `CanSyncProductFromExternalApi` |
| Test method (service) | `Should{Behavior}` / `Should{Behavior}When{Condition}` | `ShouldBeAvailableWhenPublished` |
| Test method (validator error) | `ShouldHaveErrorWhen{Scenario}` | `ShouldHaveErrorWhenEmailIsNullOrEmpty` |
| Test method (validator success) | `ShouldNotHaveErrorWhen{Scenario}` | `ShouldNotHaveErrorWhenEmailIsCorrectFormat` |
| Test method (factory) | `Prepare{Model}ShouldPopulatePropertiesFrom{Source}` | `PrepareProductModelShouldPopulatePropertiesFromEntity` |
| Test namespace | `Nop.Tests.Plugins.{Group}.{PluginName}.{Layer}` | `Nop.Tests.Plugins.Misc.MyPlugin.Services` |

---

## Common Patterns

### Resolving Services from DI Container

```csharp
var service = GetService<IMyService>();
var settings = GetService<MySettings>();
var repository = GetService<IRepository<MyEntity>>();
```

### Working with Sample Data

Sample data is installed during `BaseNopTest` initialization. Reference defaults via `NopTestsDefaults`:

```csharp
var admin = await _customerService.GetCustomerByEmailAsync(NopTestsDefaults.AdminEmail);
// NopTestsDefaults.AdminEmail = "test@nopCommerce.com"
// NopTestsDefaults.AdminPassword = "test_password"
// NopTestsDefaults.HostIpAddress = "127.0.0.1"
```

### Seeding and Cleaning Up Test Data

```csharp
[OneTimeSetUp]
public async Task SetUp()
{
    _testEntity = new MyEntity { Name = "Test", Active = true };
    await _myService.InsertEntityAsync(_testEntity);
}

[OneTimeTearDown]
public async Task TearDown()
{
    await _myService.DeleteEntityAsync(_testEntity);
}
```

### Mocking Plugin Dependencies

When a plugin service depends on an external API or SKD, use Moq for that specific dependency only. All nopCommerce services remain real:

```csharp
private Mock<IExternalApiClient> _externalApiMock;

[OneTimeSetUp]
public void SetUp()
{
    _externalApiMock = new Mock<IExternalApiClient>();
    _externalApiMock.Setup(x => x.GetDataAsync()).ReturnsAsync(new ExternalData());

    _myService = new MyPluginService(
        GetService<IRepository<MyEntity>>(),
        GetService<ILocalizationService>(),
        _externalApiMock.Object);
}
```

### Assertion Cheat Sheet

```csharp
// Equals
result.Should().Be("expected value");
result.Should().BeTrue();
result.Should().BeGreaterThan(0);

// Null / Empty
result.Should().BeNull();
result.Should().NotBeNull();
result.Should().BeNullOrEmpty();

// Collections
items.Should().HaveCount(3);
items.TotalCount.Should().Be(5);

// Validator
validator.TestValidate(model).ShouldHaveValidationErrorFor(x => x.Property);
validator.TestValidate(model).ShouldNotHaveValidationErrorFor(x => x.Property);
result.IsValid.Should().BeFalse();

// Exceptions
var ex = Assert.Throws<MyException>(() => _service.DoSomething());
```

---

## Guardrails

- **NEVER** mock nopCommerce services or `IRepository<T>` -- use the real implementations from the DI container
- **NEVER** mock Linq2DB or the database layer -- use the real SQLite in-memory database
- **NEVER** use `new Mock<IWorkContext>()` -- use `GetService<IWorkContext>()`
- **ONLY** use Moq for plugin-specific external dependencies (API clients, SDKs, file systems)
- **ALWAYS** extend `BaseNopTest` directly or via `ServiceTest` / `WebTest` -- never create your own test container
- **ALWAYS** use `[TestFixture]` on every test class
- **ALWAYS** use `GetService<T>()` to resolve services -- never resolve from `IServiceProvider` directly
- **ALWAYS** use FluentAssertions for assertions -- never use `Assert.AreEqual()` or `Assert.IsTrue()`
- **ALWAYS** add `[assembly: LevelOfParallelism(1)]` in AssemblyInfo.cs (tests run single-threaded with shared DB)
- **ALWAYS** keep test methods small and focused -- one logical assertion group per test
- **NEVER** skip the test project reference to `Nop.Tests.csproj` -- it provides `BaseNopTest` and all test infrastructure
- **NEVER** reference `Nop.Web.csproj` directly from a plugin test project -- reference `Nop.Tests` which already references `Nop.Web`
