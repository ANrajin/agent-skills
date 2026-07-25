---
name: contexts
description: Create and maintain domain context documents (technical reference files) for nopCommerce plugin projects.
---

# Domain Context Authoring

The agent MUST read this skill when generating or updating domain context documents. All documentation for a plugin lives under `docs/{PluginName}/`.

## When to Use

Activate this skill when:
- Authoring domain context documents with schemas, pipelines, and service details
- Writing a new `contexts/{domain-name}.md` file
- Updating an existing context document
- Scaffolding a new `docs/{ProjectName}/contexts/` tree
- Documenting cross-cutting concerns in `plugin-configuration.md`

## How to Use

### Step 1 — Identify the Domain Scope

Determine:
- The plugin/project name (creates the root folder under `docs\`)
- The domain areas that need context documents (e.g., product-sync, order-sync, plugin-configuration)
- Which context is cross-cutting vs. feature-specific
- Which feature specs (SPEC.md files) this context supports

### Step 2 — Create the Folder Structure

Each plugin/project gets one subfolder:

```
docs/
  {ProjectName}/
    contexts/                 # Technical reference (1 per domain)
      {domain-name}.md        # kebab-case filename
```

Naming rules:

| Item | Convention | Example |
|---|---|---|
| Project folder | PascalCase | `BusinessCentralConnector` |
| Context file | kebab-case `.md` | `product-sync.md` |

### Step 3 — Create the Context Doc (`contexts/{domain-name}.md`)

A context file is the authoritative technical reference — table schemas, service signatures, SP behaviors. It describes **how** the system is built, complementing the SPEC.md which describes **what** it must do. Adapt sections to the domain:

```markdown
# {Domain Title}

## Pipeline

```
{ASCII flow diagram with arrows}
```

## Domain Entity

### {EntityName}
| Column | Type | Notes |
|---|---|---|
| `ColumnName` | SQL_TYPE | Description |

### {PayloadName}
Deserialized from `{TableName}.Payload`.

| Field | Source |
|---|---|
| `FieldName` | JSON `path` |

## Scheduled Tasks

| Task | Type | Default Interval |
|---|---|---|
| `{TaskClassName}` | Fetch/Promote | {N} min |

### {TaskClassName}
1. {Step description}
2. {Step description}

## Services

### {InterfaceName} / {ImplementationName}
- `MethodSignature` — description
- Error handling behavior
```

---

#### Pipeline
- ASCII art with `→` arrows showing complete data flow
- Include triggering event, scheduled tasks, staging steps, promotion steps
- Show the full lifecycle from external trigger to final state

#### Domain Entity
- Full SQL table schema: Column, Type, Notes
- All columns with types, constraints, defaults
- Payload class mapping: Field, Source (JSON path)
- Backtick-quoted identifiers

#### Scheduled Tasks
- Table with class name, type (Fetch/Promote), default interval in minutes
- Numbered-step descriptions below the table for each task

#### Services
- Each service gets a subsection: `### {InterfaceName} / {ImplementationName}`
- Method signatures and responsibilities as bullets
- Error handling behavior (e.g., "throws X → session marked Y")

#### Additional Sections
Add as needed based on the domain:

| Section | When to Include |
|---|---|
| **Settings** | Domain has configurable settings (table of property, type, default) |
| **Authentication** | Inbound middleware and/or outbound OAuth flows |
| **Stored Procedure** | Domain uses SPs (step-by-step with deployment notes) |
| **Infrastructure** | DI registration, middleware pipeline Order values |
| **Permissions** | Domain-specific permission records |
| **Key Files** | Important source files for this domain (path + description) |

### Step 4 — Cross-Reference with Feature Specs

- Context files must match the feature specs (SPEC.md) they support
- If a spec references domain behavior, the corresponding context must document the implementation detail
- Cross-reference entities (like a mapping table) must be documented in all relevant contexts
- Reference the spec's requirement IDs when documenting behavior that implements a specific requirement

---

## Common Patterns

### Cross-Cutting `plugin-configuration.md`

The plugin configuration context is always cross-cutting and consolidates:

| Concern | Goes In |
|---|---|
| Settings model (properties, types, defaults) | `contexts/plugin-configuration.md` |
| Admin UI card groupings and view files | `contexts/plugin-configuration.md` |
| Authentication (inbound middleware, outbound OAuth) | `contexts/plugin-configuration.md` |
| Shared services (e.g., session status PATCH) | `contexts/plugin-configuration.md` |
| DI, middleware, route registration with Order values | `contexts/plugin-configuration.md` |
| Permissions and admin menu | `contexts/plugin-configuration.md` |
| Localization pattern | `contexts/plugin-configuration.md` |

If authentication or session management warrant their own feature specs, create those specs under `features/` but do **not** create separate context files — keep their technical depth in `plugin-configuration.md`.

### When to Create a New Context vs. Extend an Existing One

| Scenario | Action |
|---|---|
| New domain area (e.g., order-sync alongside product-sync) | Create a new `contexts/{domain}.md` |
| New entity within an existing domain | Add a section to the existing context |
| Cross-cutting concern (auth, settings, DI) | Add to `plugin-configuration.md` |
| Implementation detail for a feature spec | Add to the domain context that matches the feature |

---

## Guardrails

### Structure & Naming
- ALWAYS create the folder structure before writing context files
- ALWAYS use kebab-case for context file names
- ALWAYS place all documentation under `docs/{PluginName}/`
- ONE context file per domain — do not fragment a domain across multiple files

### Content Standards
- ALWAYS backtick-quote identifiers: column names, field names, class names, method names
- ENSURE full table schemas include Column, Type, and Notes for every column
- ALWAYS use `—` (em-dash) separator between method signature and description
- ALWAYS include ASCII pipeline diagrams when the domain has a data flow
- ENSURE numbered steps for scheduled task descriptions

### Relationship to Specs
- Context files describe **how** — feature specs (SPEC.md) describe **what**
- NEVER duplicate requirement statements in context files — reference the spec's REQ IDs instead
- ENSURE every context file corresponds to at least one feature spec
- ALWAYS update the context when the corresponding spec changes

### Cross-Cutting Concerns
- ALWAYS consolidate cross-cutting concerns into `plugin-configuration.md`
- NEVER create separate context files for authentication, settings, or DI — those go in `plugin-configuration.md`
- If a concern spans multiple domains, document it in `plugin-configuration.md` and reference it from domain-specific contexts
