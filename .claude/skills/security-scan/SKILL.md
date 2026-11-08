---
name: security-scan
description: >-
  Perform an AUTHORIZED defensive security review ("فحص الموقع") of the DAW by
  Dalilin merchant web app in this repo. Use when the user asks to scan, audit,
  review, or pentest this site/app for vulnerabilities. Covers the client-side
  index.html (XSS, CSP, token storage, input handling) and the contract with the
  Google Apps Script backend. Defensive and scoped to assets the user owns.
---

# Security Scan — DAW by Dalilin (merchant)

A repeatable checklist for reviewing the security of this app. This skill is for
**authorized** testing only: the app and its Google Apps Script backend belong to
the repository owner. Do not use it against third-party systems.

## Scope

- **In scope (code we have):** `index.html` — the entire single-page client:
  markup, inline CSS, inline JS, CSP meta tag, and how it talks to the backend.
- **Out of band (code we do NOT have):** the two Google Apps Script web apps:
  - `USERS_API`  → login / signup / OTP verify (Code 1, users sheet)
  - `PRODUCTS_API` → save / list / delete products (Code 2, products sheet)
  Their URLs are, by design, public. **All real authorization, rate-limiting,
  OTP brute-force protection and input validation must live there.** Always
  remind the owner to review that Apps Script code separately — the client
  cannot enforce any of it.

## How to run the scan

1. **Read `index.html` fully.** Don't rely on grep alone.
2. Walk the checklist below, section by section.
3. For each finding record: location (`index.html:line`), severity
   (Critical/High/Medium/Low/Info), concrete impact, and a fix.
4. Produce a short Markdown report, most severe first. Never invent findings —
   if the backend decides something, say "depends on backend, cannot verify."
5. Only test against the owner's own deployment. No scanning of other domains.

## Checklist

### 1. Cross-Site Scripting (XSS)
- Every dynamic value written via `innerHTML` (`render()`, `plist`) must pass
  through `esc()` or a validator (`imgSrc()`). Flag any interpolation that does
  not. Check product fields, titles, image `src`, and any server-returned text.
- Confirm `esc()` covers `& < > " '`.
- Backend-returned strings (name, company, product data) rendered in the DOM are
  attacker-influenced if the backend doesn't sanitize — trace each one.

### 2. Content-Security-Policy (`<meta http-equiv>`)
- `script-src 'unsafe-inline'` means any successful HTML/JS injection executes.
  Recommend moving inline scripts to hashed (`'sha256-...'`) or nonce sources and
  dropping `'unsafe-inline'`.
- Verify `default-src 'none'`, `base-uri`, `form-action`, and `connect-src` are
  still tight and only list the Google Apps Script hosts actually used.

### 3. Token & session handling
- The bearer `token` is kept in `localStorage.daw_session` → readable by any
  injected script (XSS → full account takeover). Note this coupling with §1/§2.
- Check `credentials:'omit'`, `referrerPolicy:'no-referrer'` are preserved on
  `fetch` so the token is never leaked cross-site.
- On logout, confirm `daw_session`, `s`, and session storage are cleared.

### 4. Authentication / OTP flow
- `verify` trusts a client-supplied `id` (`authId`). The backend MUST bind the
  OTP to the server session that issued it, else one user could verify another's
  code. Flag as "verify server-side."
- OTP is 6 numeric digits → backend must rate-limit and expire codes
  (`toomany`, `session` errors suggest it does — confirm).

### 5. Account enumeration / info disclosure
- Signup `exists` error reveals whether an email is registered.
- Login `choose` / `chooseAcc()` returns `{company, name, id}` for an email.
  Confirm the backend only returns this **after** the password is validated,
  otherwise it leaks account/company/name. Flag either way.

### 6. Input validation & file upload
- Client validation (`REQ`, `pct`) is UX only — backend must re-validate all
  fields, lengths, price numerics, and array sizes.
- Images: `imgSrc()` allows only Google hosts or `data:image/jpeg;base64,...`.
  Confirm the regex can't be bypassed (e.g. `javascript:` URLs) and that size is
  capped (currently 2 images, scaled to 800px, jpeg .72).

### 7. Autocomplete & minor hygiene
- Password field uses `autocomplete="current-password"` even in signup mode;
  signup should use `new-password`.
- Check for any `eval`, `document.write`, `Function()`, or dynamic script
  injection (none expected).

## Reporting template

```
# Security Scan — DAW by Dalilin
Date: <today> · Scope: index.html (client) · Backend: not reviewed

## Summary
<one line + counts by severity>

## Findings
### [SEVERITY] <title> — index.html:<line>
Impact: ...
Fix: ...

## Backend follow-up (owner action)
- Review USERS_API / PRODUCTS_API Apps Script for authz, OTP binding,
  rate-limiting, and server-side validation.
```
