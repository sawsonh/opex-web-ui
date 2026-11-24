---
parent: high-level-design
prefix: ACCESS
---

# Access Requests

## Context and Design Philosophy

There is no self sign-up. A person who wants an editor account asks for one through a public form; `opex-web-api` stores the request and emails the administrator, who reads requests in the UI and creates accounts in Cognito. Access Requests owns the public form and the administrator's read-only list.

Principles:

- **Reveal nothing.** The form's response is the same whenever the API accepts a request, whether the email already has an account, a request is already pending, or the honeypot was filled. The form is not a way to probe who uses OpEx.
- **Reachable from everywhere, guarded by the API.** Anyone can submit, in any session state, with no `Authorization` header. Throttling, spam handling, and validation are the API's.
- **Submitted text is data, never markup.** What a requester types is shown to the administrator as plain text.

## Public Form

Route: `/request-access`, linked from the landing page, the demo indicator, the sign-in pages, and the "no access yet" page.

| Field | Input | Client validation |
|---|---|---|
| Email | `type="email"`, `autocomplete="email"` | Required; email shape |
| Name | text, `autocomplete="name"` | Required |
| Company | text, `autocomplete="organization"` | Required |
| Role at the company | text, `autocomplete="organization-title"` | Required |
| Team | text, labeled "Team you want access to" | Optional |
| Reason | textarea | Required |
| Honeypot (`website`) | Hidden text input, off-screen, `tabindex="-1"`, `aria-hidden="true"`, `autocomplete="off"` | None; submitted as is |

- The API's access-request schema fixes the final field set and any length limits; limits declared in the OAS appear through Forms' counters.
- When a signed-in user opens the form, Email is pre-filled from their session and remains editable.
- Submission is `POST /v1/access-requests` with no `Authorization` header in every session state (API Client rule). The form uses Forms' confirmed-save flow, keeps no drafts, and is disabled while submitting.
- **Accepted** (any 2xx): the form is replaced by "Thanks — your request has been sent. If it's approved, you'll receive an email with a temporary password." with a link back to where the user came from. The confirmation does not echo the submitted values.
- **Validation errors** (400/422): shown against fields; input kept.
- **Rate limited** (429): "Too many requests — try again in {n} minutes" (or "later" without `Retry-After`); input kept.
- **Network or server failure**: input kept, retry offered.

## Administrator's List

Route: `/admin/access-requests`, shown only when `can(user, 'readAccessRequests')`. It is not tenant-scoped. Area descriptor: `id: 'access-requests'`, group `administration`, capability `readAccessRequests`, `availableInDemo: false`, `usesDateRange: false`; lazy-loaded so non-administrators never download it.

```
Access requests                                          [Search…]
┌──────────────────┬─────────────┬──────────────┬──────────┬───────────────────────┐
│ Submitted        │ Name        │ Email        │ Company  │ Team · Role · Reason  │
├──────────────────┼─────────────┼──────────────┼──────────┼───────────────────────┤
│ 23 Nov, 09:14    │ Dana Smith  │ dana@… [Copy]│ Acme     │ EM · We'd like to… ▸  │
└──────────────────┴─────────────┴──────────────┴──────────┴───────────────────────┘
                                                    ‹ Newer   Older ›
```

- Newest first. Paging follows the API's paging; the UI never loads every request to page or filter locally.
- Search is sent to the API when it supports it; without API search, the search box is not shown.
- Submitted time is shown in the administrator's time zone.
- The reason is truncated to one line and expands in place.
- "Copy" copies the email for creating the user in Cognito.
- Every requester-supplied value is rendered as plain text: no HTML, no Markdown, no auto-linking of URLs.
- The list is read-only. Approving means creating the user in Cognito, outside the UI.

## Layers

| Layer | Contents |
|---|---|
| `domain/` | `AccessRequestDraft` (form values), `AccessRequest` (submitted, as listed) |
| `application/` | `AccessRequestRepository` port: `submit(draft)`, `list(page, search?)` |
| `infrastructure/` | Adapter over the generated client; the submit call is made with the public, unauthenticated request type |
| `ui/` | Public form page, confirmation, administrator's list |

## Decisions & Alternatives

| Decision | Chosen | Alternatives Considered | Rationale |
|---|---|---|---|
| Spam control in the UI | Honeypot field only; everything else at the API | CAPTCHA; client-side throttling | A CAPTCHA needs a third-party script, which the Content Security Policy excludes; client-side throttling is trivially bypassed. The API is where throttling can actually hold. |
| Response to accepted requests | One identical confirmation | Tell the user if they already have an account or a pending request | Distinct responses would let anyone test whether an email is known to OpEx. |
| Rendering requester text | Plain text only | Markdown; auto-linked URLs | The administrator is the most privileged user; rendering untrusted text as anything but text invites injection and phishing links. |
| Pre-filling email when signed in | Pre-filled, editable | Not pre-filled; locked | Saves typing for an editor requesting another tenant, without assuming they want the same address. |

## Open Questions & Future Decisions

### Deferred
1. Whether requests carry statuses the administrator can set (HLD open question). The list is read-only until that is decided.
2. The API's paging and search support for the list.

## References

- `docs/high-level-design.md` — Access requests; Non-Goals (managing users)
- `docs/intent/forms/forms-design.md` — confirmed save; no drafts for this form
- `docs/intent/api-client/api-client-design.md` — unauthenticated `POST /v1/access-requests`
