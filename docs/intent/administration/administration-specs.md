# Tenant Administration — EARS Specs

Design: `administration-design.md`

## Settings: access and layout

- [ ] **ADMIN-SET-001**: The Settings area descriptor shall have id `tenant-administration`, group `administration`, capability `editTenantConfiguration`, `availableInDemo: false`, and `usesDateRange: false`, at `/t/{tenantId}/settings`.
- [ ] **ADMIN-SET-002**: The Settings page shall show one section per source system in the registry, in registry order, each a separate form with key `tenant-config/{sourceSystem}` and its own save controls.
- [ ] **ADMIN-SET-003**: The Settings page header shall show the tenant's name and type.
- [ ] **ADMIN-SET-004**: When the tenant's type is `DEMO`, the Settings page shall show above its sections "This is a public demo tenant. Anyone can view its reporting data and insights without signing in."
- [ ] **ADMIN-SET-005**: The source-system registry shall declare GitHub (repository, `owner/name`, `^[A-Za-z0-9-]+/[A-Za-z0-9._-]+$`), SonarQube (project key, `^[A-Za-z0-9_.:-]+$`), Jira (project key, `^[A-Z][A-Z0-9_]+$`), and PagerDuty (service ID, `^P[A-Z0-9]{6}$`), each with its label, item noun, identifier hint, and identifier format.

## Settings: editing a source system

- [ ] **ADMIN-EDIT-001**: Each Settings section shall list the source system's configured items by identifier, with the last-collected time in the browser's time zone and any collection error, when the API provides them.
- [ ] **ADMIN-EDIT-002**: When the user enters an identifier and chooses "Add" in a Settings section, the section shall add it to the list marked "New" if it matches the source system's identifier format and is not already listed.
- [ ] **ADMIN-EDIT-003**: If an identifier entered in a Settings section fails the source system's identifier format, or is already listed (compared case-insensitively for GitHub and exactly otherwise), then the section shall not add it and shall show the reason under the input.
- [ ] **ADMIN-EDIT-004**: Each Settings section shall show "{n} of 10 selected" for its source system, and while 10 items are listed, shall disable adding with "Remove an item to add another".
- [ ] **ADMIN-EDIT-005**: When the user removes an item in a Settings section, the section shall mark it for removal without sending a request until the user saves.
- [ ] **ADMIN-EDIT-006**: When the user saves a Settings section whose changes remove items, the section shall first ask for confirmation listing them — "Collection will stop for: {items}. Data already collected stays visible." — and send the request only if the user confirms.
- [ ] **ADMIN-EDIT-007**: When the user saves a Settings section whose changes only add items, the section shall send the request without a confirmation.
- [ ] **ADMIN-EDIT-008**: When the API rejects a Settings save because an identifier does not exist or is not accessible in its source system, the section shall show the error against that item and keep every other value.
- [ ] **ADMIN-EDIT-009**: When the API confirms a Settings save, the section shall show the configuration the API returned and, when the API provides it, when the change takes effect.
- [ ] **ADMIN-EDIT-010**: A Settings section shall allow saving with zero items.

## Tenants (administrator)

- [ ] **ADMIN-TEN-001**: The Tenants area descriptor shall have id `tenants`, group `administration`, capability `manageTenants`, `availableInDemo: false`, and `usesDateRange: false`, at `/admin/tenants`, loaded lazily.
- [ ] **ADMIN-TEN-002**: The Tenants page shall list every tenant from the API's tenant list, sorted by name, showing name, type, and tenant id with a copy button, each name linking to that tenant's Settings.
- [ ] **ADMIN-TEN-003**: Wherever Tenant Administration shows a tenant's type, it shall be read-only, with a note that the type is changed only by hand in MongoDB Atlas.
- [ ] **ADMIN-TEN-004**: The New tenant form shall require a name, and a type chosen from `Customer`, `Sandbox`, and `Demo` with no option preselected; submitting without a type shall show "Choose a tenant type".
- [ ] **ADMIN-TEN-005**: When the administrator submits the New tenant form with type `Demo`, the form shall first ask "Create a public demo tenant? Anyone will be able to view {name}'s reporting data and insights without signing in." and send the request only if confirmed.
- [ ] **ADMIN-TEN-006**: When the API confirms a new tenant, the Tenants page shall show its name, type, and tenant id, the instruction to create the Cognito group `tenant-{tenantId}` in the main user pool and add the tenant's editors to it, a copy button for the group name, and a link to the tenant's Settings.

## Layers

- [ ] **ADMIN-LAYER-001**: The Tenant Administration domain layer (source-system registry, items, configuration, tenants, tenant types, the 10-item limit) shall have no dependency on React or HTTP.
