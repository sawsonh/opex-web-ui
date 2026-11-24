---
parent: high-level-design
prefix: FORM
---

# Forms

## Context and Design Philosophy

The UI writes tenant configuration, anomaly feedback, improvement tickets, access requests, and insights run requests. Forms is the shared machinery every one of those writes goes through: form state, client-side validation display, the confirmed-save flow, presenting rejections and failures, the unsaved-changes guard, and drafts that survive a session ending.

Principles:

- **Saved means the API said so.** The form shows "Saved" only after the API confirms, and then shows the record the API returned, not the values the user typed. (HLD tenet: confirmed over optimistic.)
- **Never lose the user's input.** A rejected or failed save keeps every value. Leaving with unsaved changes asks first. A session ending keeps a draft to resume.
- **The API validates; the form helps.** Client-side checks cover shape and required fields for fast feedback. The API's verdict is final and is shown against the fields it names.

## Building a Form

Areas use one hook and a few components:

```ts
useConfirmedForm<Values, Saved>({
  formKey: string;                         // stable id, e.g. 'tenant-config/github'
  schema: ZodType<Values>;                 // shape and required fields only
  initialValues: Values;                   // from the API record, or blank for create
  submit: (values: Values) => Promise<Saved>;   // the area's mutation, via its adapter
  toValues: (saved: Saved) => Values;      // how the confirmed record maps back to the form
  fieldPathForApiError?: (apiField: string) => FieldPath<Values> | undefined;
  sensitiveFields?: FieldPath<Values>[];   // never written to drafts
})
```

Built on React Hook Form (state, field registration) with a Zod resolver (client validation) and a TanStack Query mutation (submit). Components: `Form`, `Field` (label, input, description, error wiring), `ErrorSummary`, `SaveBar` (submit, status, retry).

## Save Flow

```
editing ──submit──▶ client validation ──fails──▶ editing (errors shown, focus first error)
                          │ passes
                          ▼
                     submitting (submit disabled, "Saving…")
          ┌───────────────┼──────────────────────┬────────────────────────┐
     2xx  ▼          validation (400/422)    conflict (409)        network / server / rateLimited / forbidden
       saved            rejected               conflict                    failed
 reset to toValues   field + summary       "changed since you      message, input kept,
 (API record),       errors, input kept    opened it", input kept   "Retry" resubmits the
 invalidate queries,                                                 same values (not for
 draft deleted,                                                      forbidden)
 "Saved"
```

- Only one submission is in flight per form; submit is disabled while `submitting`.
- **Saved**: the form resets to `toValues(saved)` so what is displayed is the API's record. The area's mutation invalidates the queries the write affects, and other views refetch. A polite live region announces "Saved".
- **Rejected**: API field errors are placed on fields via `fieldPathForApiError`; errors it cannot place appear in the `ErrorSummary` at the top of the form. The summary lists every error as a link to its field, and focus moves to it.
- **Conflict**: the form keeps the user's values and offers "Load latest version", which, after confirmation, replaces the form values with the current record from the API. Nothing is merged automatically.
- **Failed**: the message depends on the `ApiError` kind — network ("Couldn't reach OpEx"), server ("OpEx couldn't save this"), rate limited ("Too many requests — try again in {duration}" with the wait in seconds or minutes and retry disabled until then, or "Too many requests — try again later" when the API gives no `Retry-After`), forbidden ("Your account isn't permitted to do this", no retry).
- `unauthorized` does not reach the form: Auth & Session ends the session and the draft carries the input across sign-in.

## Length and Count Limits

Business-rule limits (HLD § Business rules) are part of a form's Zod schema, using the area's domain constant.

- A text field with a length limit shows a live counter ("3,950 / 4,000") once the text reaches 80% of the limit. Length is counted in Unicode code points.
- The field does not use the HTML `maxlength` attribute, so pasting long text never silently drops the excess. Text over the limit stays as typed, the counter and field show the error ("{n} characters over the limit"), and submit is blocked until it is shortened.
- A multi-select with a count limit shows "{n} of {limit} selected"; at the limit, unselected options are disabled with the reason, and selected ones can still be removed.

## Unsaved-Changes Guard

- A form is *dirty* when its values differ from the last confirmed values.
- Navigating away inside the app (React Router blocker) while dirty asks "Discard unsaved changes?" with "Stay" and "Discard".
- Closing or reloading the tab while dirty triggers the browser's `beforeunload` prompt.
- The guard does not fire after a successful save, and does not fire for the redirect to sign-in when a session ends — the draft covers that case.

## Drafts

- While a form is dirty, its values are written to the tab's `sessionStorage`, debounced by 500 ms, under `opex:draft:{userId}:{tenantId | '-'}:{formKey}`. Fields listed in `sensitiveFields`, and any `type="password"` input, are omitted.
- On mount, if a draft exists for the same key and differs from `initialValues`, the form restores it and shows "Restored your unsaved changes" with "Discard draft".
- A draft is deleted when the form saves, when the user discards it, and when they confirm leaving with unsaved changes.
- Signing out deletes all drafts in the tab (`clearAllDrafts()`, called by Auth & Session). A session that ends by expiry or rejection keeps them.
- Storage reads and writes are wrapped so a full or unavailable `sessionStorage` disables drafts without breaking the form.
- Drafts are written only in the signed-in state. Demo and `none` sessions have no user id and write none, and the public access-request form never writes a draft in any state, so people sharing a browser cannot see each other's unsent requests.

## Accessibility

- Every input has a visible label; descriptions and errors are linked with `aria-describedby`; invalid inputs set `aria-invalid`.
- Status changes (saving, saved, failed) are announced through a live region.
- Errors are text, never color alone.

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Client schema library | Zod via the React Hook Form resolver | Yup; hand-written validators | Typed schemas infer the form's `Values` type, so schema and type cannot diverge; Zod is the most common pairing with React Hook Form. |
| Displayed values after save | Reset to the API's returned record | Keep the submitted values | The API may normalize or default fields; showing its record is the only way "Saved" means what is stored. |
| Conflict resolution | Keep input; explicit "Load latest version" | Auto-merge; last write wins silently | A silent merge or overwrite could change what the collector does without anyone noticing. |
| Draft storage | `sessionStorage` per tab, keyed by user, tenant, form | `localStorage`; in memory | Survives the redirect to sign-in and a reload without outliving the tab or leaking to the next person on a shared machine. |
| Drafts on sign-out | Deleted | Kept | Signing out is a deliberate end; the next person to use the tab must not see the previous user's input. |

## Open Questions & Future Decisions

### Deferred
1. The API's validation error body, which determines how `fieldPathForApiError` receives field names.
2. Whether records carry a version or ETag; if they do, a restored draft whose base version is older than the current record shows the conflict notice before the user saves.

## References

- `docs/high-level-design.md` — Writes; Key Design Decision 9; Tenets
- `docs/intent/api-client/api-client-design.md` — `ApiError` kinds
- `docs/intent/auth-session/auth-session-design.md` — session end, sign-out, `UserContext`
