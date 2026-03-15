### Postmortem: Frontend unauthenticated GraphQL handling

#### What happened

- After protecting `money/graphql` with `login_required`, unauthenticated requests from the SPA sometimes received HTML login redirects or 302→200 HTML bodies instead of JSON. The frontend Apollo client was configured with a simple `uri` and no global error handling, so these cases surfaced as parse/network errors and were not redirecting the user to login.

#### Root causes

- Apollo client did not send `credentials: 'include'`, so session cookies were not included.
- No global error link: 401/403 responses and HTML bodies were not mapped to an auth flow.
- Django login redirect returns HTML; without detection, Apollo attempts to parse JSON and fails.

#### Fix

- Switched to `createHttpLink` with `credentials: 'include'` and a custom `fetch` wrapper to detect redirects or HTML content and force a login redirect.
- Added `onError` link to handle GraphQL `UNAUTHENTICATED` codes and HTTP 401/403.

#### How to avoid next time

- Always include credentials when using session-auth with a different-origin SPA.
- Add a global error link early in setup for auth and network handling.
- When securing endpoints behind Django auth, plan the SPA redirect path (`/accounts/login/?next=...`) and detect HTML fallback responses.

#### Follow-ups

- Add user-friendly toasts for other network errors.
- Add E2E test: unauthenticated user visiting a data page gets redirected to login.
