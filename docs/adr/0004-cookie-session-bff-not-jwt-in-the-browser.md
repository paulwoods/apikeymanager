# Cookie session (BFF), not a JWT in the browser

Status: accepted

Operators authenticate via OIDC, but the flow runs entirely server-side in Spring Security.
The browser receives an `httpOnly`, `Secure`, `SameSite=Lax` session cookie and never sees
a token; the React app calls the API with `credentials: 'include'`.

The conventional alternative — the SPA completes the OIDC flow, holds a JWT in
`localStorage`, and sends it as a bearer token — was rejected because `localStorage` is
readable by any script on the origin. Any XSS anywhere in the app would exfiltrate a
credential granting full administrative control over every API key in the estate. For a
general CRUD app that trade is arguable; for the console that issues and revokes
credentials it is not.

## Consequences

CSRF protection is now required, because cookies are attached automatically. Spring
Security's CSRF tokens are enabled, and the Vite dev server proxies the API so cookie
origins match in development.

Verifier services cannot use this mechanism — a cookie session suits browsers, not
machines. They authenticate by OAuth2 client-credentials and present a bearer JWT, so the
application runs **two disjoint Spring Security filter chains**: `/api/**` matched to the
cookie session, `/internal/**` matched to a JWT resource server. Keeping them disjoint is
what prevents privilege bleeding between the operator and verifier audiences.
