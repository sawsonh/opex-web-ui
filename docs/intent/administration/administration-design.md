---
parent: high-level-design
prefix: ADMIN
---

# Tenant Administration

## Context and Design Philosophy

Tenant Administration is where editors, for their tenants, and the administrator, for any tenant, choose which source-system items `opex-web-api` records in the tenant configuration and `opex-collector-worker` collects: GitHub repositories, SonarQube projects, Jira projects, PagerDuty services, and future source systems. It configures *what* is collected, never *how* the collector authenticates. It is also where the administrator lists every tenant and creates new ones, each with a name and a tenant type.

Principles:

- **Configuration changes what the collector does.** Every save goes through the confirmed-save flow in Forms, removals are confirmed explicitly, and nothing is shown as saved until the API returns it.
- **Type it; the API checks it.** Editors enter an item's identifier directly. The UI checks its format; `opex-web-api` checks the item exists in the source system before storing it, and the UI shows the API's verdict against the item.
- **Each source system stands alone.** Source systems are configured and saved independently, so an edit to GitHub repositories never conflicts with a concurrent edit to Jira projects.
- **Publishing is deliberate.** A `DEMO` tenant's reporting data and insights are public. Choosing that type is an explicit, confirmed act, and members working in a demo tenant are always told it is public.

## Route and Access

- `/t/{tenantId}/settings`, shown in navigation as "Settings" to users for whom `can(user, 'editTenantConfiguration', tenant)` holds. Not available in the demo.
- Area descriptor: `id: 'tenant-administration'`, group `administration`, capability `editTenantConfiguration`, `availableInDemo: false`, `usesDateRange: false`.
- `/admin/tenants`, shown in navigation as "Tenants" to users for whom `can(user, 'manageTenants')` holds — the administrator only. It is not tenant-scoped. Area descriptor: `id: 'tenants'`, group `administration`, capability `manageTenants`, `availableInDemo: false`, `usesDateRange: false`; lazy-loaded so other users never download it.

## Page Layout

```
Settings · acme
┌─ GitHub repositories ──────────────────────────── 4 of 10 selected ─┐
│ acme/api            Last collected 22 Nov 2026, 14:05     [Remove]   │
│ acme/web            Collection failing: repository not accessible    │
│ …                                                                    │
│ [ owner/name                                    ]  [Add]             │
│                                          [Discard]  [Save changes]  │
└──────────────────────────────────────────────────────────────────────┘
┌─ SonarQube projects ──────────────────────────── 2 of 10 selected ─┐
└──────────────────────────────────────────────────────────────────────┘
┌─ Jira projects ─┐  ┌─ PagerDuty services ─┐   …
```

One section per source system, in a fixed order. Each section is its own form (`formKey: tenant-config/{sourceSystem}`) with its own save bar.

The page header shows the tenant's name and type, or "—" while the type is unknown. For a tenant known to be `DEMO`, a banner above the sections reads "This is a public demo tenant. Anyone can view its reporting data and insights without signing in."

## Source-System Registry

Source systems are declared in the module's domain as data, so adding one adds an entry, its adapter mapping, and its API support — not new page logic.

```ts
type SourceSystem = {
  id: 'github' | 'sonarqube' | 'jira' | 'pagerduty' | string;
  label: string;          // 'GitHub'
  itemNoun: string;       // 'repository' / 'repositories'
  identifierHint: string; // shown under the input, e.g. 'owner/name'
  identifierFormat: RegExp;
};
const MAX_ITEMS_PER_SOURCE_SYSTEM = 10;      // HLD § Business rules
```

| Source system | Identifier | Format checked in the UI |
|---|---|---|
| GitHub | Repository, `owner/name` | `^[A-Za-z0-9-]+/[A-Za-z0-9._-]+$` |
| SonarQube | Project key | `^[A-Za-z0-9_.:-]+$` |
| Jira | Project key | `^[A-Z][A-Z0-9_]+$` |
| PagerDuty | Service ID | `^P[A-Z0-9]{6}$` |

These are shape checks for fast feedback only; whether the item exists, and whether the collector can reach it, is the API's check.

## Editing a Source System

- **Current items** show the item's identifier and, when the API provides them, its last-collected time (in the user's time zone) and any collection error the API reports for it.
- **Adding**: a text input with the source system's identifier hint and an "Add" button. An identifier that fails the format check, or is already in the list (compared case-insensitively for GitHub; exactly otherwise), is not added and the reason is shown under the input. Added items appear in the list marked "New" until saved.
- **The limit**: the section header shows "{n} of 10 selected"; at 10, the add control is disabled with "Remove an item to add another". The API enforces the same limit.
- **Removing** marks the item for removal in the form; nothing changes until save.
- **Saving**: when the change removes items, a confirmation lists them — "Collection will stop for: acme/legacy, acme/old-api. Data already collected stays visible." — before the request is sent. Additions alone save without a confirmation.
- **Items the API rejects**: when the API reports that an identifier does not exist or is not accessible, the error appears against that item, every other value is kept, and nothing is saved until the item is corrected or removed.
- **After save**: the section shows the API's returned configuration and the API's statement of when the change takes effect, when it provides one.
- **Conflict**: if another editor saved the same source system in the meantime, Forms' conflict flow applies ("Load latest version").
- An empty source system is valid: saving zero items stops collection from that source system for the tenant.

## Tenants (Administrator)

```
Tenants                                                     [New tenant]
┌──────────────────────┬──────────┬──────────────────────────┐
│ Name                 │ Type     │ Tenant ID                │
├──────────────────────┼──────────┼──────────────────────────┤
│ Acme                 │ Customer │ 6b0480d6e2f8…  [Copy]    │
│ Acme demo            │ Demo     │ 6b0480d6e2f9…  [Copy]    │
└──────────────────────┴──────────┴──────────────────────────┘
```

- **List**: every tenant from the API's tenant list, sorted by name, showing name, type, and tenant id with a copy button. Each name links to that tenant's Settings. The same list feeds the administrator's tenant switcher.
- **Type is read-only.** Wherever a tenant's type is shown, it cannot be edited; a note explains that a tenant's type is changed only by hand in MongoDB Atlas, because switching a tenant to `DEMO` publishes its data.
- **New tenant** opens a form (`formKey: tenants/new`) with:
  - **Name**: required; length limit from the OAS when it declares one, through Forms' counter.
  - **Type**: required radio group with no option preselected — `Customer`, `Sandbox` ("behaves like Customer; for testing"), and `Demo` ("its reporting data and insights are visible to anyone without signing in"). Submitting without a choice shows "Choose a tenant type".
- **Creating a `DEMO` tenant** asks for confirmation before the request is sent: "Create a public demo tenant? Anyone will be able to view {name}'s reporting data and insights without signing in."
- **After creation** (confirmed by the API), the administrator's tenant list is refreshed, so the shell's switcher and tenant checks include the new tenant, and the page shows the new tenant's name, type, and id, and the remaining manual step: "Create the Cognito group `tenant-{tenantId}` in the main user pool and add the tenant's editors to it", with a copy button for the group name, and a link to the new tenant's Settings.

## Layers

| Layer | Contents |
|---|---|
| `domain/` | `SourceSystem` registry, `SourceItem`, `TenantSourceConfiguration`, `MAX_ITEMS_PER_SOURCE_SYSTEM`, the rule computing which items a change removes; `Tenant` (id, name, type) and `TenantType` (`DEMO`, `CUSTOMER`, `SANDBOX`) |
| `application/` | `TenantConfigurationRepository` port: `load(tenant)`, `save(tenant, sourceSystem, items)`; `TenantRepository` port: `list()`, `create(name, type)` |
| `infrastructure/` | Adapter over the generated client; maps per-item collection status and the API's conflict, limit, and per-item existence errors |
| `ui/` | Settings page, one `useConfirmedForm` per source system, removal confirmation, demo banner; Tenants page and New tenant form |

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| How items are entered | Typed identifier, format-checked in the UI, existence-checked by the API on save | Picking from a list of candidates the API fetches from each source system | Typing needs no listing endpoint per source system; the API's check before storing still prevents configuring an item that does not exist. |
| Save unit | One source system at a time | The whole tenant configuration in one save | Smaller saves conflict less, and a failure in one source system does not block the others. |
| Removal confirmation | Required when a save removes items | None; confirm every save | Removal stops collection, which is the consequential change; confirming additions too would train people to click through. |
| Per-item status | Shown when the API provides it | Not shown | Editors need to see that a configured item is actually being collected, not only that it is configured. |
| Tenant type on creation | Required choice, no default; confirmation for `DEMO` | Default to `CUSTOMER` | A default makes the choice easy to skip; `DEMO` publishes data, so it must be chosen and confirmed deliberately. |
| Changing a tenant's type | Not offered in the UI | An edit control | The API does not allow it; the change is a manual act in MongoDB Atlas so that publishing a tenant's data is never one click away. |

## Open Questions & Future Decisions

### Deferred
1. When a saved change takes effect in collection.
2. How the API reports a per-item existence failure (so it can be placed against the item).
3. Whether records carry a version or ETag for conflict detection (HLD open question).
4. Whether the API reports per-item collection status (last collected, errors).
5. The tenant list and tenant creation endpoints, and the `name` length limit, from `opex-web-api`'s OAS.

## References

- `docs/high-level-design.md` — Business rules; Writes; Roles and tenants
- `docs/intent/forms/forms-design.md` — confirmed save, count limits, conflicts
- `docs/intent/app-shell/app-shell-design.md` — area descriptor, navigation
