---
name: views
description: Create and manage Razor views (.cshtml) in nopCommerce plugins following the admin layout, card containers, search panels, data tables, localization, tag helpers, and styling standards.
---

# nopCommerce Plugin Views Skill

## When to Use

Activate this skill when:

- Building or modifying the configuration page (`Configure.cshtml`) for a plugin.
- Creating list pages featuring search panels and nopCommerce DataTables (Kendo wrappers).
- Designing Create, Edit, or Details forms using nopCommerce tag helpers (`nop-editor`, `nop-select`, etc.).
- Adding scripts, AJAX interactions, or styles to views.
- Restructuring views using tag helpers and collapsible card components.
- Setting up general view import templates (`_ViewImports.cshtml`) and view starts (`_ViewStart.cshtml`).

---

## How to Use

### Step 1 — Configure `_ViewImports.cshtml` and `_ViewStart.cshtml`

All plugin view directories should contain a `_ViewImports.cshtml` and a `_ViewStart.cshtml` to ensure standard layouts and tag helpers are available. Every admin view must use `_AdminLayout` by default.

**`_ViewStart.cshtml`:**
```html
@{
    Layout = "_AdminLayout";
}
```

**`_ViewImports.cshtml`:**
```html
@inherits Nop.Web.Framework.Mvc.Razor.NopRazorPage<TModel>

@using Microsoft.AspNetCore.Mvc.Rendering
@using Nop.Web.Framework.Models
@using Nop.Web.Framework.Models.DataTables
@using Nop.Web.Framework.Mvc.Routing
@using Nop.Web.Framework.UI
@using Nop.Web.Framework

@inject Nop.Services.Common.IGenericAttributeService GenericAttributeService
@inject Nop.Web.Framework.UI.INopHtmlHelper NopHtml
@inject Nop.Web.Framework.Mvc.Routing.INopUrlHelper NopUrl
@inject Nop.Core.IWorkContext WorkContext

@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
@removeTagHelper Microsoft.AspNetCore.Mvc.TagHelpers.InputTagHelper, Microsoft.AspNetCore.Mvc.TagHelpers
@addTagHelper *, Nop.Web.Framework

@using Nop.Plugin.Misc.MyPlugin.Models
```
*Note: `removeTagHelper` for `InputTagHelper` is critical so nopCommerce's custom tag helpers handle inputs correctly.*

---

### Step 2 — Use nopCommerce Tag Helpers for Form Elements

Always use nopCommerce tag helpers instead of standard HTML inputs to ensure consistent styling and model binding behavior.

```html
<div class="form-group row">
    <div class="col-md-3">
        <!-- Optional: Multi-store override checkbox -->
        <nop-override-store-checkbox asp-for="MyField_OverrideForStore" asp-input="MyField" asp-store-scope="@Model.ActiveStoreScopeConfiguration" />
        <nop-label asp-for="MyField" />
    </div>
    <div class="col-md-9">
        <!-- Use nop-editor for text/numeric inputs -->
        <nop-editor asp-for="MyField" />
        <!-- Use nop-select for dropdowns -->
        <nop-select asp-for="MyDropdownId" asp-items="Model.AvailableDropdownItems" />
        <span asp-validation-for="MyField"></span>
    </div>
</div>
```

---

### Step 3 — Build Configuration Pages (`Configure.cshtml`)

Configuration pages require specific setups for multi-store configuration, active menu items, and collapsible cards. **Crucially, the configuration view must override the default layout by setting `Layout = "_ConfigurePlugin";`.**

```html
@model ConfigurationModel
@{
    Layout = "_ConfigurePlugin";
    ViewBag.PageTitle = T("Plugins.MyPlugin.Configuration.Title").Text;
    
    // Set the active menu item in the left sidebar
    NopHtml.SetActiveMenuItemSystemName("Plugins.MyPlugin.Configuration");

    var customer = await WorkContext.GetCurrentCustomerAsync();
    const string hideGeneralBlockAttributeName = "MyPlugin.HideGeneralBlock";
    var hideGeneralBlock = await GenericAttributeService.GetAttributeAsync<bool>(customer, hideGeneralBlockAttributeName);
}

<form asp-controller="MyPlugin" asp-action="Configure" method="post">
    <div class="content-header clearfix">
        <h1 class="float-left">
            @T("Plugins.MyPlugin.Configuration.Title")
            <small>
                <i class="fas fa-arrow-circle-left"></i>
                <a asp-controller="Plugin" asp-action="List" asp-route-area="Admin">
                    @T("Admin.Configuration.Plugins.Misc.BackToList")
                </a>
            </small>
        </h1>
        <div class="float-right">
            <button type="submit" name="save" class="btn btn-primary">
                <i class="far fa-save"></i>
                @T("Admin.Common.Save")
            </button>
        </div>
    </div>

    <section class="content">
        <div class="container-fluid">
            <div class="form-horizontal">
                <!-- Multi-store configuration component -->
                @await Component.InvokeAsync(typeof(StoreScopeConfigurationViewComponent))
                <input type="hidden" asp-for="ActiveStoreScopeConfiguration" />
                
                <nop-cards id="my-plugin-panels">
                    <nop-card asp-name="my-plugin-general" 
                              asp-icon="fas fa-gear" 
                              asp-title="@T("Plugins.MyPlugin.Configuration.GeneralSettings")" 
                              asp-hide-block-attribute-name="@hideGeneralBlockAttributeName" 
                              asp-hide="@hideGeneralBlock" 
                              asp-advanced="false">
                        @await Html.PartialAsync("_Configure.GeneralSettings", Model)
                    </nop-card>
                </nop-cards>
            </div>
        </div>
    </section>
</form>
```

---

### Step 4 — Implement Search Panels and DataTables for List Pages

List pages combine a search panel (which toggles via generic attributes) and a `DataTablesModel`.

```html
@model MySearchModel
@{
    ViewBag.PageTitle = T("Plugins.MyPlugin.List.Title").Text;
    NopHtml.SetActiveMenuItemSystemName("Plugins.MyPlugin.List");

    const string hideSearchBlockAttributeName = "MyPluginList.HideSearchBlock";
    var hideSearchBlock = await GenericAttributeService.GetAttributeAsync<bool>(await WorkContext.GetCurrentCustomerAsync(), hideSearchBlockAttributeName);
}

<form asp-controller="MyPlugin" asp-action="List" method="post">
    <!-- Header Omitted for Brevity -->
    
    <section class="content">
        <div class="container-fluid">
            <div class="form-horizontal">
                <div class="cards-group">
                    
                    <!-- Search Panel -->
                    <div class="card card-default card-search">
                        <div class="card-body">
                            <div class="row search-row @(!hideSearchBlock ? "opened" : "")" data-hideAttribute="@hideSearchBlockAttributeName">
                                <div class="search-text">@T("Admin.Common.Search")</div>
                                <div class="icon-search"><i class="fas fa-search" aria-hidden="true"></i></div>
                                <div class="icon-collapse"><i class="far fa-angle-@(!hideSearchBlock ? "up" : "down")" aria-hidden="true"></i></div>
                            </div>
                            <div class="search-body @(hideSearchBlock ? "closed" : "")">
                                <div class="row">
                                    <div class="col-md-6">
                                        <div class="form-group row">
                                            <div class="col-md-4"><nop-label asp-for="SearchName" /></div>
                                            <div class="col-md-8"><nop-editor asp-for="SearchName" /></div>
                                        </div>
                                    </div>
                                </div>
                                <div class="row">
                                    <div class="col-md-12 text-center">
                                        <button type="button" id="search-entities" class="btn btn-primary btn-search">
                                            <i class="fas fa-search"></i>
                                            @T("Admin.Common.Search")
                                        </button>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>

                    <!-- DataTable Card -->
                    <nop-cards id="entities-grid-panels">
                        <nop-card asp-name="entities-grid-panel"
                                  asp-icon="fas fa-list"
                                  asp-title="@T("Plugins.MyPlugin.List")"
                                  asp-hide-block-attribute-name="MyPluginList.HideGridBlock"
                                  asp-hide="false"
                                  asp-advanced="false">
                            <div class="card-body">
                                @await Html.PartialAsync("Table", new DataTablesModel
                                {
                                    Name = "entities-grid",
                                    UrlRead = new DataUrl("ListAction", "MyPlugin", null),
                                    SearchButtonId = "search-entities",
                                    Length = Model.PageSize,
                                    LengthMenu = Model.AvailablePageSizes,
                                    Filters = new List<FilterParameter>
                                    {
                                        new FilterParameter(nameof(MySearchModel.SearchName))
                                    },
                                    ColumnCollection = new List<ColumnProperty>
                                    {
                                        new ColumnProperty(nameof(MyEntityModel.Name))
                                        {
                                            Title = T("Plugins.MyPlugin.Fields.Name").Text,
                                            Width = "200"
                                        },
                                        new ColumnProperty(nameof(MyEntityModel.Status))
                                        {
                                            Title = T("Plugins.MyPlugin.Fields.Status").Text,
                                            Width = "100",
                                            Render = new RenderCustom("renderStatus") // JS Function
                                        },
                                        new ColumnProperty(nameof(MyEntityModel.Id))
                                        {
                                            Title = T("Admin.Common.View").Text,
                                            Width = "100",
                                            ClassName = NopColumnClassDefaults.CenterAll,
                                            Render = new RenderCustom("renderViewButton")
                                        }
                                    }
                                })
                            </div>
                        </nop-card>
                    </nop-cards>
                </div>
            </div>
        </div>
    </section>
</form>

<script asp-location="Footer">
    function renderStatus(data, type, row, meta) {
        var statusId = row.StatusId;
        var badgeClass = statusId === 10 ? 'badge badge-info' : 'badge badge-success';
        var textRenderer = $.fn.dataTable.render.text().display;
        return '<span class="' + badgeClass + '">' + textRenderer(data) + '</span>';
    }
    
    function renderViewButton(data, type, row, meta) {
        return '<a class="btn btn-default" onclick="viewDetails(' + row.Id + ');return false;" href="#"><i class="far fa-eye"></i>@T("Admin.Common.View").Text</a>';
    }
</script>
```

---

## Common Patterns

### Pattern 1 — Handling AJAX Calls with Anti-Forgery Tokens

When writing custom AJAX scripts within Razor views, use the `addAntiForgeryToken` function (provided by nopCommerce) to append the verification token to the request payload:

```javascript
var postData = { id: id };
addAntiForgeryToken(postData);

$.ajax({
    method: 'POST',
    url: '@Url.Action("ActionName", "ControllerName")',
    data: postData,
    dataType: 'json',
    success: function (result) {
        // Handle result
    }
});
```

### Pattern 2 — Showing Alerts via `nop-alert`
Use the `<nop-alert>` tag helper and the `showAlert` JS function for showing dynamic notifications in views (e.g., after an AJAX call):

```html
<!-- At the bottom of the view -->
<nop-alert asp-alert-id="my-custom-alert" />

<script>
    // Inside a JS function
    showAlert('my-custom-alert', 'Operation successful!');
</script>
```

---

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

</Project>
```

**Rule:** Always use glob patterns for cshtml files — never list each cshtml individually.
Do not add `<None Remove="...">` entries for cshtml files; the glob `Content Include` handles them.

## Guardrails
- Controller `action` and View file name must be same. Never add a different viewfile name, unless explicitly defined.
- **Guard Clauses:** Always validate input arguments at method entry using C# 11 style guard clauses. Use `ArgumentNullException.ThrowIfNull(x)` rather than conditional checks throwing exception.
- **Admin Layouts:** Every admin view must use `_AdminLayout` (defined in `_ViewStart.cshtml`), except for `Configure.cshtml` which MUST explicitly use `Layout = "_ConfigurePlugin";`.
- **Form Placement:** NEVER place `<form>` elements directly inside a `<nop-card>` as direct children. Doing so breaks the javascript collapse/expand behavior. Wrap card fields within partial view files nested inside the card instead.
- **Input Tag Helpers:** Ensure `@removeTagHelper Microsoft.AspNetCore.Mvc.TagHelpers.InputTagHelper, Microsoft.AspNetCore.Mvc.TagHelpers` is in `_ViewImports.cshtml`. Always use `<nop-editor>`, `<nop-select>`, and `<nop-label>`. Do not use standard `<input class="form-control">` elements unless for very specific custom UI (like a read-only display field).
- **No Hardcoded Strings:** Every text label, button, placeholder, and hint must be localized using `@T("Resource.String.Key")`. No hardcoded plaintext strings are allowed. For `PageTitle` or DataTable Columns, use `.Text` (e.g., `T("...").Text`).
- **State Constants:** Always declare page hide block variables and attribute names as constant strings at the top of the Razor view (e.g., `const string hideSearchBlockAttributeName = "PageName.HideSearchBlock"`). Do not inline magic strings.
- **Menu Active State:** Always set `NopHtml.SetActiveMenuItemSystemName("System.Name")` in the main view to ensure the left navigation highlights correctly.
- **Async Calls:** Always use `await` and asynchronous execution when fetching user attributes or calling service methods inside views. (e.g., `await GenericAttributeService.GetAttributeAsync(...)`).
- **No Back Button on Lists:** Never add a back button to list view pages (it's acceptable on Configuration or Edit pages, but not main grid lists).
- **ViewBag Limitations:** Use `ViewBag.PageTitle` for setting the page title. Never use it to pass core business or domain data. Use model factories and view models.
