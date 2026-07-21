# Tasks

Implementation order, with status. Terminology in [CONTEXT.md](../CONTEXT.md), decisions in
[DESIGN.md](../DESIGN.md) and [docs/adr](./adr).

Ordering principle: **ApiKey before authentication.** ApiKey is the product — nothing else
has value without it, and building it will apply design pressure to the rest of the model
that is better discovered before auth is hardened. Deferring auth is cheap here because the
seam is a single `AuditorAware` bean, not a change spread through the codebase. It is
**not** cheap to defer past deployment: see task 12.

---

## Phase 1 — Foundations

| # | Task | Status |
|---|---|---|
| 1 | Design interview; glossary in `CONTEXT.md`, decisions in `DESIGN.md` | **Complete** |
| 2 | ADRs 0001–0004 (digest choice, terminal revocation, cascade, cookie/BFF) | **Complete** |
| 3 | Three public repos wired as git submodules | **Complete** |
| 4 | Backend scaffold — Spring Boot 4.1.0, Java 25, Postgres, Flyway, Testcontainers, compose | **Complete** |
| 5 | Frontend scaffold — Vite 8, React 19, TS, Tailwind v4, TanStack Query, RHF/Zod, Vitest/RTL/MSW | **Complete** |

## Phase 2 — Tenancy

| # | Task | Status |
|---|---|---|
| 6 | Application: entity, `V1`/`V2`, partial unique index, REST, soft delete | **Complete** |
| 7 | Application UI: list, create, withdraw | **Complete** |
| 8 | KeyOwner: entity, `V3`, tenant-scoped partial index, nested REST, cascade (ADR-0003) | **Complete** |
| 9 | KeyOwner UI: application detail page, nested route | **Complete** |

## Phase 3 — ApiKey (the product)

| # | Task | Status |
|---|---|---|
| 10 | **ApiKey domain** — `V4` migration, issuance, revocation, expiry | **Pending** |
| 11 | **Verification endpoint** — the hot path | **Pending** |
| 12 | **ApiKey UI** — issuance, one-time reveal, revocation | **Pending** |

### 10. ApiKey domain

- `V4`: `api_key` table — id, application_id, key_owner_id, key_digest, key_prefix,
  status, expires_at, audit columns. **No `deleted_at`** (ADR-0002).
- Unique index on `key_digest`; index supporting single-row verification lookup.
- Generation: 32 bytes from `SecureRandom`, Base64url unpadded, `akm_` prefix.
- Digest: SHA-256. Plaintext returned **only** from the issuance response — never stored,
  never logged, never retrievable (ADR-0001).
- `status` enum `ACTIVE`/`REVOKED`; revocation **terminal**, no un-revoke.
- `expires_at` NOT NULL; default 90 days, configurable ceiling 365 days, rejected beyond.
- Inject `Clock` — deferred during Application deliberately, and now genuinely required to
  test expiry without sleeping.
- Extend the cascade one level: withdrawing a KeyOwner revokes its keys; withdrawing an
  Application cascades through its KeyOwners to their keys, one transaction (ADR-0003).
- Listing exposes Key Prefix only.

### 11. Verification endpoint

- Single indexed lookup on digest — `status = 'ACTIVE' AND expires_at > now()`, no joins.
- Constant-time digest comparison.
- Identical response shape **and timing** for absent / revoked / expired.
- The presented key is never logged, at any level, even truncated.
- Expiry derived at read time; no scheduled job, no stored flag.

### 12. ApiKey UI

- Issue form: choose KeyOwner, set expiry within the ceiling.
- **One-time reveal** — explicit acknowledgment before dismissal, copy-to-clipboard, plain
  statement that it will not be shown again. The plaintext exists only in the create
  response; a refresh loses it permanently.
- List shows Key Prefix, status, and expiry as a relative countdown.
- Action is **Revoke**, not delete, with confirmation — it is irreversible.

## Phase 4 — Authentication

**Blocks deployment. Nothing in phases 1–3 is deployable without this.**

| # | Task | Status |
|---|---|---|
| 13 | Keycloak in Docker Compose | **Pending** |
| 14 | Operator OIDC via BFF — server-side flow, httpOnly cookie, CSRF (ADR-0004) | **Pending** |
| 15 | Real `AuditorAware` from the security context — retire `STUB-OPERATOR-AUTH-NOT-WIRED` | **Pending** |
| 16 | Verifier auth — client-credentials JWT, second filter chain over `/internal/**` | **Pending** |
| 17 | Frontend: 401 handling and login redirect | **Pending** |

Today `/api/**` has **no authentication whatsoever** — anything that can reach port 8080
has full administrative control over every application, key owner and key.

## Phase 5 — Hardening

| # | Task | Status |
|---|---|---|
| 18 | Rate limiting on the verification endpoint | **Pending** |
| 19 | Audit that no code path logs a presented key | **Pending** |
| 20 | Security review of the whole surface | **Pending** |

## Phase 6 — Gaps found along the way

Real but non-blocking; each was noticed during earlier work and consciously deferred.

| # | Task | Status |
|---|---|---|
| 21 | Validation errors return a generic `"Invalid request content."` with no field detail — unhelpful to direct API callers | **Pending** |
| 22 | Application rename UI — `PATCH` exists on the backend, nothing in the UI calls it | **Pending** |
| 23 | Pagination UI — the API pages at 20, the UI only ever shows the first page | **Pending** |
| 24 | CI — no pipeline exists; both suites run only on a developer machine | **Pending** |
| 25 | READMEs — all three repos still carry placeholder content | **Pending** |

## Deferred by design

Recorded in `DESIGN.md`, deliberately out of scope until asked for.

| Task | Status |
|---|---|
| Key rotation UX — issue replacement with an overlap window before revoking the old key | **Pending** |
| Expiry warning notifications | **Pending** |
| Last-used tracking on keys — useful for finding dead keys, costs a write per verification | **Pending** |
| Scopes / permissions on keys | **Pending** |
