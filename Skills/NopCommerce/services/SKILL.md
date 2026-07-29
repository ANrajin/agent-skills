---
name: services
description: >-
  Create business-logic service classes (interface + implementation) for a
  nopCommerce plugin, registered via DI in NopStartup and injected into
  controllers/factories rather than queried directly. Use when adding a new
  service to a plugin's Services/ folder.
---

# Services in nopCommerce

## When to Use

Activate this skill when:
- Adding a new business-logic class to a plugin's `Services/` folder
- Adding CRUD, search/paged-list, or bulk upsert methods for a domain entity
- Wrapping an external API call (HTTP integration) behind a service
- Adding a cache-backed read method for reference/lookup data
- Deciding whether a new method belongs in a service vs. a controller or model factory (it belongs in the service — controllers and model factories must not query repositories directly)

---

## Core Conventions

- One interface per service: `I{Name}Service` and `{Name}Service`, both in namespace `{PluginNamespace}.Services` (or a sub-namespace like `.Services.Auth` for grouped concerns).
- Follow the [[class-structure]] region layout: `Fields` → `Ctor` → `Utilities` → `Methods`. Private helpers go in `Utilities`; public API goes in `Methods`.
- Constructor-inject every dependency into a `readonly` field — never resolve services with a service locator.
- Every public method is async and suffixed `Async`, returning `Task`/`Task<T>`.
- Register both interface and implementation as scoped in the plugin's `Infrastructure/PluginNopStartup.cs`:
  ```csharp
  services.AddScoped<IMyEntityService, MyEntityService>();
  ```
- Never inject `ILogger<T>` from `Microsoft.Extensions.Logging` — use `Nop.Services.Logging.ILogger` (`_logger.InformationAsync/WarningAsync/ErrorAsync`).
- Never hardcode user-facing or logged strings — resolve them via `ILocalizationService.GetResourceAsync("Plugin.X.Y.Z")` (see [[localization]]).

---

## Data Access

Inject `IRepository<TEntity>` directly — do not write raw SQL or use `DbContext`.

```csharp
public class MyEntityService : IMyEntityService
{
    #region Fields

    private readonly IRepository<MyEntity> _myEntityRepository;

    #endregion

    #region Ctor

    public MyEntityService(IRepository<MyEntity> myEntityRepository)
    {
        _myEntityRepository = myEntityRepository;
    }

    #endregion

    #region Methods

    public async Task<MyEntity> GetByIdAsync(int id)
    {
        return await _myEntityRepository.GetByIdAsync(id, null);
    }

    public async Task InsertAsync(MyEntity entity)
    {
        await _myEntityRepository.InsertAsync(entity);
    }

    public async Task UpdateAsync(MyEntity entity)
    {
        await _myEntityRepository.UpdateAsync(entity);
    }

    public async Task DeleteAsync(MyEntity entity)
    {
        await _myEntityRepository.DeleteAsync(entity);
    }

    #endregion
}
```

Common repository/queryable members: `.Table` (IQueryable), `.GetByIdAsync(id, cacheKeyFunc)`, `.GetAllAsync(query => ...)`, `.InsertAsync(entity|list)`, `.UpdateAsync(entity|list)`, `.DeleteAsync(entity|list)`, and the `ToListAsync()` / `CountAsync()` / `ToPagedListAsync(pageIndex, pageSize)` extensions from `Nop.Core`/`Nop.Data`.

### Search / Paged-List Pattern

Build the query incrementally from optional filter parameters, then page it:

```csharp
public async Task<IPagedList<MyEntity>> SearchAsync(
    string name = null,
    int? statusId = null,
    int pageIndex = 0,
    int pageSize = 100)
{
    var query = _myEntityRepository.Table.AsQueryable();

    if (!string.IsNullOrWhiteSpace(name))
        query = query.Where(e => e.Name.Contains(name));

    if (statusId.HasValue)
        query = query.Where(e => e.StatusId == statusId.Value);

    query = query.OrderByDescending(e => e.CreatedOnUtc);

    return await query.ToPagedListAsync(pageIndex, pageSize);
}
```

### Bulk Upsert Pattern

Look existing rows up in one query, split into inserts/updates, then batch-write:

```csharp
public async Task<int> BulkUpsertAsync(IList<(string key, string payload)> items)
{
    if (items == null || !items.Any())
        return 0;

    var keys = items.Select(i => i.key).Distinct().ToList();
    var existingLookup = (await _myEntityRepository.Table
        .Where(e => keys.Contains(e.Key))
        .ToListAsync())
        .ToDictionary(e => e.Key);

    var inserts = new List<MyEntity>();
    var updates = new List<MyEntity>();

    foreach (var (key, payload) in items)
    {
        if (existingLookup.TryGetValue(key, out var existing))
        {
            existing.Payload = payload;
            updates.Add(existing);
        }
        else
        {
            inserts.Add(new MyEntity { Key = key, Payload = payload });
        }
    }

    if (inserts.Any())
        await _myEntityRepository.InsertAsync(inserts);
    if (updates.Any())
        await _myEntityRepository.UpdateAsync(updates);

    return items.Count;
}
```

---

## Caching Reference/Lookup Data

Cache read-mostly data, not volatile/staging/session tables. Define keys in a `{Name}Defaults` (or `{Name}CacheDefaults`) static class:

```csharp
public static class MyEntityDefaults
{
    public static CacheKey MyEntityCacheKey => new("Nop.myplugin.myentity.{0}-{1}");
    public static string MyEntityCachePrefix => "Nop.myplugin.myentity.";
}
```

Read through `IStaticCacheManager` (or `IShortTermCacheManager` for request-scoped caching), and invalidate the prefix on every write:

```csharp
private readonly IStaticCacheManager _staticCacheManager;

public async Task<IList<MyEntity>> GetAllCachedAsync()
{
    return await _staticCacheManager.GetAsync(
        MyEntityDefaults.MyEntityCacheKey,
        async () => await _myEntityRepository.GetAllAsync(q => q));
}

public async Task UpdateAsync(MyEntity entity)
{
    await _myEntityRepository.UpdateAsync(entity);
    await _staticCacheManager.RemoveByPrefixAsync(MyEntityDefaults.MyEntityCachePrefix);
}
```

For entity-change-driven invalidation instead of doing it inline on every write, add a `Services/Cache/{Entity}CacheEventConsumer.cs` subclassing `CacheEventConsumer<TEntity>` and override `ClearCacheAsync`.

---

## External API Integration Services

For services that call out to a third-party API:

- Inject `IHttpClientFactory` and call `CreateClient()` per request; set an explicit `Timeout`.
- Build requests with `HttpRequestMessage` rather than the convenience `Get/PostAsync` overloads when you need custom headers (e.g. bearer tokens).
- On `401 Unauthorized`, re-acquire the token once and retry — don't loop indefinitely.
- On a non-success status, read the response body, log it via `ILogger.ErrorAsync`, and throw (typically `HttpRequestException`) so the caller can mark its own state as failed.
- Catch specific exception types at the call site (e.g. `HttpRequestException`, `InvalidOperationException`) to update session/status state and log — don't swallow errors silently.

```csharp
var httpClient = _httpClientFactory.CreateClient();
httpClient.Timeout = TimeSpan.FromMinutes(5);

var request = new HttpRequestMessage(HttpMethod.Get, url);
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);

var response = await httpClient.SendAsync(request);

if (response.StatusCode == HttpStatusCode.Unauthorized)
{
    token = await _tokenService.AcquireTokenAsync();
    request = new HttpRequestMessage(HttpMethod.Get, url);
    request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
    response = await httpClient.SendAsync(request);
}

if (!response.IsSuccessStatusCode)
{
    var errorBody = await response.Content.ReadAsStringAsync();
    await _logger.ErrorAsync($"MyApi returned {(int)response.StatusCode}: {errorBody}");
    throw new HttpRequestException($"MyApi returned {(int)response.StatusCode}");
}
```

---

## Related: Scheduled Tasks

`IScheduleTask` classes (`Nop.Services.ScheduleTasks`) are the usual caller of long-running services (e.g. a fetch/sync service). They are registered as concrete scoped types (not by interface):

```csharp
services.AddScoped<MyEntityFetchTask>();
```

Unlike controllers, tasks **do** wrap their body in `try/catch` — there is no MVC pipeline to catch and log an unhandled exception for you:

```csharp
public async Task ExecuteAsync()
{
    try
    {
        var settings = await _settingService.LoadSettingAsync<MyPluginSettings>();
        if (!settings.Enabled)
            return;

        // ... call services
    }
    catch (Exception ex)
    {
        await _logger.ErrorAsync(await _localizationService.GetResourceAsync("Plugin.MyPlugin.Task.ExecutionError"), ex);
    }
}
```

Load settings via `ISettingService.LoadSettingAsync<T>()` inside a task (so it always sees the latest saved values); a service can instead have the settings POCO constructor-injected directly when it only needs the value at request time.

---

## Guardrails

- **No repository access outside services** — controllers, model factories, and view components call services, never `IRepository<T>` directly.
- **No manual SQL / raw ADO.NET** — express queries through the repository's `IQueryable` (`.Table`) and Linq2DB extensions.
- **No `Microsoft.Extensions.Logging.ILogger<T>`** — always `Nop.Services.Logging.ILogger`.
- **No hardcoded user-facing or logged strings** — always resolve via `ILocalizationService`.
- **No caching of volatile/staging/session data** — only cache stable reference/lookup data, and always invalidate by prefix on write.
- **Don't swallow exceptions** — log via `ILogger.ErrorAsync` and either rethrow or set explicit failure state on the caller's record/session.
- **Keep services thin per concern** — one service per aggregate/entity family; compose multiple services in a task or controller rather than growing one god-service.
