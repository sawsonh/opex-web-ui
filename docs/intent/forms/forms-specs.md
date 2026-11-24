# Forms — EARS Specs

Design: `forms-design.md`

## Validation

- [ ] **FORM-VAL-001**: When the user submits a form, `useConfirmedForm` shall validate the values against the form's Zod schema and, if validation fails, show the errors without sending a request.
- [ ] **FORM-VAL-002**: When a form shows validation errors, it shall list every error in an `ErrorSummary` at the top of the form, each linking to its field, and move focus to the summary.
- [ ] **FORM-VAL-003**: Each form input shall have a visible label, link its description and error with `aria-describedby`, and set `aria-invalid` while invalid.

## Save flow

- [ ] **FORM-SAVE-001**: While a form's submission is in flight, the form shall disable its submit control, show "Saving…", and send no second submission.
- [ ] **FORM-SAVE-002**: When the API confirms a save, the form shall reset its values to `toValues` of the record the API returned, announce "Saved" through a polite live region, and let the area's mutation invalidate the affected queries.
- [ ] **FORM-SAVE-003**: The form shall never show "Saved" before the API has confirmed the save.
- [ ] **FORM-SAVE-004**: When a save is rejected as `validation`, the form shall keep every value, show each API field error on the field that `fieldPathForApiError` maps it to, and list errors it cannot map in the `ErrorSummary`.
- [ ] **FORM-SAVE-005**: When a save is rejected as `conflict`, the form shall keep every value, state that the record changed since it was opened, and offer "Load latest version", which replaces the form's values with the API's current record only after the user confirms.
- [ ] **FORM-SAVE-006**: If a save fails as `network` or `server`, then the form shall keep every value, show "Couldn't reach OpEx" or "OpEx couldn't save this" respectively, and offer "Retry", which resubmits the same values.
- [ ] **FORM-SAVE-007**: If a save fails as `rateLimited`, then the form shall keep every value and, when `retryAfterSeconds` is known, show "Too many requests — try again in {duration}" (in seconds below one minute, otherwise in whole minutes rounded up) and disable "Retry" until that time has passed; when it is not known, show "Too many requests — try again later".
- [ ] **FORM-SAVE-008**: If a save fails as `forbidden`, then the form shall keep every value, show "Your account isn't permitted to do this", and offer no retry.
- [ ] **FORM-SAVE-009**: The form shall never merge the API's current record into the user's values automatically.

## Length and count limits

- [ ] **FORM-LIMIT-001**: A text field with a business-rule length limit shall count length in Unicode code points.
- [ ] **FORM-LIMIT-002**: While a length-limited text field holds at least 80% of its limit, it shall show a live counter "{n} / {limit}".
- [ ] **FORM-LIMIT-003**: A length-limited text field shall not use the HTML `maxlength` attribute and shall never truncate typed or pasted text.
- [ ] **FORM-LIMIT-004**: While a length-limited text field is over its limit, it shall show "{n} characters over the limit" and the form shall block submission.
- [ ] **FORM-LIMIT-005**: A multi-select with a business-rule count limit shall show "{n} of {limit} selected".
- [ ] **FORM-LIMIT-006**: While a count-limited multi-select is at its limit, it shall disable its unselected options with the reason and keep its selected options removable.

## Unsaved-changes guard

- [ ] **FORM-GUARD-001**: While a form is dirty (its values differ from the last confirmed values), navigating elsewhere in the app shall ask "Discard unsaved changes?" with "Stay" and "Discard".
- [ ] **FORM-GUARD-002**: While a form is dirty, closing or reloading the tab shall trigger the browser's leave-page prompt.
- [ ] **FORM-GUARD-003**: The unsaved-changes guard shall not prompt after a confirmed save, nor when the session ends and the application navigates to sign-in.

## Drafts

- [ ] **FORM-DRAFT-001**: While the session is `signedIn` and a form is dirty, the form shall write its values to the tab's `sessionStorage` under `opex:draft:{userId}:{tenantId or -}:{formKey}`, debounced by 500 ms.
- [ ] **FORM-DRAFT-002**: A form draft shall omit the fields listed in `sensitiveFields` and every `type="password"` input.
- [ ] **FORM-DRAFT-003**: When a form mounts in the `signedIn` state and a draft exists for its key that differs from its initial values, the form shall restore the draft and show "Restored your unsaved changes" with "Discard draft".
- [ ] **FORM-DRAFT-004**: The form shall delete its draft when it saves, when the user discards the draft, and when the user confirms leaving with unsaved changes.
- [ ] **FORM-DRAFT-005**: The forms module shall provide `clearAllDrafts()`, which deletes every form draft in the tab.
- [ ] **FORM-DRAFT-006**: While the session is `none` or `demo`, forms shall write no drafts, and the public access-request form shall write no draft in any session state.
- [ ] **FORM-DRAFT-007**: If `sessionStorage` is unavailable or full, then forms shall stop writing drafts and continue to work without error.
