---
name: nopcommerce-plugin-manual
description: Generate a Markdown user manual for a nopCommerce plugin Rajin has just built, covering Overview/Features, Prerequisites & Compatibility, Installation, Configuration, and Usage/How-To, with screenshot placeholder markers for the visual admin-panel steps. Use this whenever Rajin asks to write, draft, or generate a "user manual", "plugin documentation", "install guide", "user guide", or "how to use" doc for a nopCommerce plugin — even if he just pastes the plugin's README, source code, config page details, or raw unstructured notes and says "turn this into the manual" or "document this plugin." Also trigger on requests to update or extend an existing manual for a plugin.
---

# nopCommerce Plugin User Manual Generator

Turns Rajin's plugin material (README, source snippets, config screens, raw notes — any mix) into a Markdown user manual with a fixed section order, so every plugin's manual has the same shape and can be dropped into a wiki or AppSource listing with minimal editing.

## Before drafting: gather what's missing

Never invent plugin behavior, config field names, or feature claims that aren't in the material Rajin provided. If the input is missing something a section needs, ask him for it directly rather than guessing plausible-sounding content — a wrong install step or config field in a user-facing manual is worse than a short one.

Specifically check the input covers:
- **Plugin name** (exact, as it appears in nopCommerce admin > Plugins) and version
- **What it does**, in plain terms (for Overview)
- **Compatibility**: supported nopCommerce version(s), any dependent plugins/services (for Prerequisites)
- **Installation**: how it's installed — via admin panel upload, NuGet, manual DLL drop, etc.
- **Configuration**: the actual settings/fields on its config page, with what each one does
- **Usage**: the concrete admin/storefront actions a user takes once it's configured
- **Admin sidebar menu location**: the exact path a user clicks through to reach the plugin (e.g. "Admin > Third-party plugins > X", "Admin > Nop-Station > X") — needed for Installation/Configuration/Usage navigation steps

If Rajin supplies source code, look for these signals rather than asking him to restate them:
- `*.csproj` / plugin description file — name, version, supported nopCommerce version
- `Controllers/*AdminController.cs` and matching `Views/**/Configure.cshtml` — the actual config fields and their labels/tooltips
- `Migrations/` or `Install()`/`Uninstall()` methods — install-time setup (schema, settings, permissions)
- `RouteProvider.cs` or route attributes — any storefront-facing endpoints (usage-relevant)
- Localization resource files (`*.resx` or `AddOrUpdatePluginLocaleResource` calls) — often contain the clearest plain-language descriptions of each feature/field, written for the admin UI

**Admin menu location is a special case — always ask, never infer.** A plugin's own `AdminMenuCreatedEventConsumer` (or equivalent) typically only proves the menu item's *title* and that it's added as a child node to some event — it does not by itself tell you which top-level menu group that child renders under, since the parent group is usually defined in a shared/core assembly (e.g. `NopStation.Core`) that won't be part of the plugin's own source. Don't fill this in by pattern-matching to a previous plugin's menu structure, and don't guess a plausible-sounding group name ("Third-party plugins", "Nop-Station", etc.) even if the plugin.json `Group` field hints at one — ask Rajin for the actual path he sees in his admin sidebar.

If after checking source + notes something else is still unclear (e.g., a config field with no obvious purpose from code alone), ask Rajin rather than guessing.

## Fixed section order

Always produce these five sections, in this order. Do not add Troubleshooting/FAQ or Support/Contact sections unless Rajin explicitly asks for them on a given manual — they're not part of the default template.

An optional sixth section, **Developer Integration**, is added after Usage/How-To only when the source material contains genuinely developer-facing extension/integration content — e.g., how another plugin or developer builds on top of this one, interfaces to implement, code samples, DI registration, assembly-reference gotchas. This section is for plugin developers, not store admins, so code and technical terms are fine here even in an otherwise merchant-facing manual. If the source material has no such content, omit this section entirely — don't add it "just in case."

### Handling source content that doesn't fit a section (e.g. "Constraints" / "Scope" lists)

Source material sometimes includes a raw list of technical constraints or an in-scope/out-of-scope list that doesn't map directly onto the five fixed sections. Triage each item:
- **Admin-relevant** (affects what a store admin should know or do — compatibility limits, uninstall/reinstall behavior, "don't do X" warnings that affect their setup) → fold into the closest matching section (usually Prerequisites & Compatibility, Installation, or Usage). Don't create a new section for these.
- **Developer-only** (implementation details: internal method signatures, type-identity/build-configuration rules, retry/threading internals) → drop from the end-user manual, unless the same information is also relevant to the Developer Integration section above, in which case it can go there instead.
- If unsure which bucket an item falls into, ask Rajin rather than guessing — don't silently include or drop something that could matter.

1. **Overview & Features** — What the plugin does, in 2-4 sentences, followed by a bullet list of its key features/capabilities.
2. **Prerequisites & Compatibility** — Supported nopCommerce version(s), any required dependent plugins/services/accounts, minimum permissions needed to install.
3. **Installation** — Numbered steps from "download/obtain the plugin" through to it appearing active in Admin > Configuration > Local Plugins. Match nopCommerce's actual install flow (upload zip or place in `/Plugins`, restart app, find & install from the plugin list) unless the material describes something different.
4. **Configuration** — Numbered steps to reach the config page (Admin > Configuration > Local Plugins > [Plugin Name] > Configure), then walk through each setting: its label, what it controls, and any constraints (required, format, default value) — pulled from the material provided, not invented.
5. **Usage / How-To** — Concrete, task-oriented steps for what an admin or customer actually does with the plugin once configured (e.g., "How to sync a product," "How to view sync status"). Use numbered steps for sequences, bullets for standalone tips.

## Screenshot placeholders

At every step where the user would be looking at a specific nopCommerce admin screen (navigating a menu, viewing a config page, seeing a result), insert a placeholder on its own line immediately after that step:

```
> 📸 **Screenshot placeholder:** [short description of what this screenshot should show]
```

Example: after "Navigate to Admin > Configuration > Local Plugins," insert:
```
> 📸 **Screenshot placeholder:** Local Plugins list showing [Plugin Name] in Not Installed state
```

Don't add placeholders for purely conceptual steps (e.g., "restart the application") where there's nothing distinct to screenshot.

## Formatting conventions

- Markdown headers: `##` for the five main sections, `###` for subsections within Configuration if there are many fields.
- Numbered lists for anything sequential (install, configure, usage steps).
- Bullet lists for features, prerequisites, and non-sequential tips.
- Config field names and code/route values in inline code formatting (`` `LikeThis` ``).
- Keep language aimed at a nopCommerce store admin/merchant audience — plain, task-oriented, minimal jargon — unless the plugin is clearly developer-facing (e.g., an API/integration connector), in which case technical terms are fine but instructions should stay actionable.

## Output

Deliver the manual as a single Markdown file. Use the plugin's exact name (as given) in the filename, e.g. `business-central-connector-user-manual.md`.
