# Access Requests — EARS Specs

Design: `access-requests-design.md`

## Public form

- [ ] **ACCESS-FORM-001**: The access-request form at `/request-access` shall be reachable in every session state, and linked from the landing page, the demo indicator, the sign-in pages, and the "no access yet" page.
- [ ] **ACCESS-FORM-002**: The access-request form shall ask for email, name, company, and role at the company (each required), the team the requester wants access to (optional), and the reason (required).
- [ ] **ACCESS-FORM-003**: The access-request form shall include a honeypot field named `website` that is visually hidden, removed from the tab order, hidden from assistive technology, and submitted as entered.
- [ ] **ACCESS-FORM-004**: While the session is `signedIn`, the access-request form shall pre-fill Email from the session and keep it editable.
- [ ] **ACCESS-FORM-005**: The access-request form shall submit `POST /v1/access-requests` without an `Authorization` header and shall write no draft, in every session state.
- [ ] **ACCESS-FORM-006**: When the API accepts an access request, the form shall be replaced by "Thanks — your request has been sent. If it's approved, you'll receive an email with a temporary password." with a link back, the same in every case and without repeating the submitted values.

## Administrator's list

- [ ] **ACCESS-LIST-001**: The access-requests area descriptor shall have id `access-requests`, group `administration`, capability `readAccessRequests`, `availableInDemo: false`, and `usesDateRange: false`, at `/admin/access-requests`, loaded lazily.
- [ ] **ACCESS-LIST-002**: The access-requests list shall show each request's submitted time in the browser's time zone, name, email with a copy button, company, team, role, and reason, newest first.
- [ ] **ACCESS-LIST-003**: The access-requests list shall page through requests using the API's paging, and shall not load every request to page or filter in the browser.
- [ ] **ACCESS-LIST-004**: Where the API supports searching access requests, the list shall show a search box that sends the search to the API; otherwise it shall show none.
- [ ] **ACCESS-LIST-005**: The access-requests list shall show each reason truncated to one line, expanding in place on request.
- [ ] **ACCESS-LIST-006**: The access-requests list shall render every requester-supplied value as plain text, with no HTML, Markdown, or automatic linking.
- [ ] **ACCESS-LIST-007**: The access-requests list shall offer no action that changes a request.
