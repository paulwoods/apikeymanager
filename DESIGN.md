# API Key Manager — Design

Decisions settled during the design interview. Terminology is defined in [CONTEXT.md](./CONTEXT.md).

## What this system is

An **issuer and verifier** of API keys. It generates keys, stores only their digests, and
answers whether a presented key is currently valid. It is on the request path of the
services that depend on it.

## Domain decisions

### Keys are hashed; plaintext exists once

- Key format: `akm_` + 32 bytes from `SecureRandom`, Base64url-encoded, unpadded.
- Stored: **Key Digest** (SHA-256 of the plaintext) + **Key Prefix** (first 8 chars, display only).
- Plaintext is returned **only** in the 201 response to issuance. It is never stored,
  never logged, and never retrievable afterwards.
- SHA-256 rather than bcrypt/Argon2 — keys carry 256 bits of entropy, so slow hashing
  defends nothing, and per-row salts would make indexed lookup impossible.
- Unique index on the digest. Verification is a single-row lookup, no joins.

### Key lifecycle is one enum plus an instant

- `status`: `ACTIVE` | `REVOKED`. No separate `active` boolean, no soft-delete on ApiKey.
- **Revocation is terminal.** There is no un-revoke; access is restored by issuing a new key.
- `expires_at` is `NOT NULL`. Default 90 days, configurable ceiling of 365 days, enforced
  by the API.
- **Expiry is always derived** by comparing `expires_at` to the clock. Never stored as a
  flag, never batch-updated.
- Verification predicate, in full:
  `status = 'ACTIVE' AND expires_at > now()`

### Soft-delete applies to Application and KeyOwner only

- Both carry `deleted_at`.
- Soft-deleting an **Application** revokes all its keys and soft-deletes its KeyOwners
  **in one transaction**. Soft-deleting a **KeyOwner** revokes its keys likewise.
- Consequence, accepted deliberately: **restoring a soft-deleted parent does not restore
  its keys.** Revocation is terminal; keys must be reissued.
- Cascading on write is what keeps verification join-free.

### Uniqueness is scoped to live rows

Partial unique indexes, because a plain `UNIQUE` counts soft-deleted rows and would let a
deleted record permanently reserve an email or name:

```sql
CREATE UNIQUE INDEX uq_key_owner_app_email
  ON key_owner (application_id, email) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX uq_application_name
  ON application (name) WHERE deleted_at IS NULL;
```

Re-adding a soft-deleted KeyOwner creates a **new row**. The old episode stays distinct
and auditable.

### Two populations of people, never conflated

- **KeyOwner** — the subject a key is issued to. A data record, scoped to one Application,
  never authenticates.
- **Operator** — a human who signs in to manage keys. Not an entity; identified in audit
  fields by an opaque string from the IdP.

### Audit fields

`date_created`, `created_by`, `date_updated`, `updated_by` on every table. The `_by`
columns are **strings**, not foreign keys, so the audit trail cannot be broken by removing
a person.

## Security

| Concern | Decision |
|---|---|
| Operator auth | OIDC via Spring Security, flow entirely server-side (BFF) |
| Browser session | `httpOnly` + `Secure` + `SameSite=Lax` cookie; **no token in `localStorage`** |
| CSRF | Spring Security CSRF tokens, required because auth rides on a cookie |
| Verifier auth | OAuth2 client-credentials; bearer JWT validated as a resource server |
| Filter chains | Two, disjoint: `/api/**` cookie session, `/internal/**` JWT |
| Digest comparison | Constant-time |
| Response uniformity | "No such key", "revoked" and "expired" are indistinguishable in shape and timing |
| Logging | The presented key is never logged, at any level, even truncated |
| Rate limiting | Applied to the verification endpoint |

Repositories are **public**, so no secret may ever enter a commit — no seeded keys, no real
configuration.

## Stack

### Backend

**Spring Boot 4.1.0** — current GA release. Requires Java 17 minimum, compatible through
Java 26. **Target Java 25 LTS**, compiled on the installed JDK 26.

Versions below are managed by the `spring-boot-dependencies:4.1.0` BOM — read from the
published POM, not assumed. Do not pin them individually.

| Dependency | Version |
|---|---|
| Spring Framework | 7.0.8 |
| Spring Security | 7.1.0 |
| Hibernate | 7.4.1.Final |
| Flyway | 12.4.0 |
| Testcontainers | **2.0.5** |
| PostgreSQL JDBC | 42.7.11 |
| Tomcat | 11.0.22 |

Two version traps, both verified against docs rather than memory:

- **Testcontainers 2.x renamed the Maven artifact.** It is
  `org.testcontainers:testcontainers-postgresql`, *not* the 1.x `org.testcontainers:postgresql`.
  The class remains `org.testcontainers.containers.PostgreSQLContainer`. Nearly every
  example online is 1.x and will fail to resolve.
- **`@UuidGenerator(style = TIME)` is UUIDv1, not v7.** Use `style = VERSION_7`.
  Confirmed available in Hibernate 7.4.

Also:

- PostgreSQL + Flyway migrations
- Errors as RFC 9457 `ProblemDetail`
- List endpoints paginated (`page` / `size`, default 20)

### Frontend

- React + Vite + **TypeScript**
- Tailwind **v4** via `@tailwindcss/vite` and `@import "tailwindcss"` —
  no `tailwind.config.js`, no PostCSS config, no `content` array
- React Router
- TanStack Query for server state and mutation-driven invalidation
- React Hook Form + Zod, with Zod as the single source of validation truth
- Calls the API with `credentials: 'include'`; Vite dev proxy so cookie origins match

### Local environment

Docker Compose: PostgreSQL + Keycloak.

## Testing — TDD, integration-weighted

Tests are written **first**.

Mocks cannot test the riskiest logic here, so it is covered against a real database:

- Partial unique indexes only fail against real Postgres
- Cascade-revoke is a transactional guarantee, including rollback on mid-way failure
- The verification predicate is the one query that fails **open** if wrong

| Layer | Tools |
|---|---|
| Backend integration | Testcontainers 2.x Postgres via `@ServiceConnection`, Flyway-migrated |
| Backend unit | Key generation, expiry-ceiling validation |
| Frontend | Vitest + React Testing Library + MSW at the network layer |

MSW intercepts HTTP rather than mocking the query client, so tests exercise the real
TanStack Query path — including that the one-time reveal renders once and is gone after
dismissal.

## Repository layout

Three **public** GitHub repos under `paulwoods`, wired as true git submodules:

| Repo | Role |
|---|---|
| `apikeymanager` | Superproject: `.gitmodules`, compose, docs |
| `apikeymanager-backend` | Submodule |
| `apikeymanager-frontend` | Submodule |

SSH URLs, matching the configured `gh` protocol.

**Accepted cost:** changes spanning the API contract are not atomic — they take a commit in
a submodule plus a pointer bump in the superproject, and the submodule must be pushed
first or clones break.

## UI constraints forced by the design

- The one-time reveal requires explicit acknowledgment before dismissal, offers
  copy-to-clipboard, and states plainly that the key will not be shown again.
- List views show the **Key Prefix only** and must never imply the full key is retrievable.
- Expiry renders as a relative countdown ("expires in 12 days"), the form in which it is
  actionable.
- "Delete key" is not offered. The action is **Revoke**, and it is permanent.

## Open — deliberately deferred

- Rotation UX (issue replacement, overlap window before revoking the old key)
- Expiry warning notifications
- Last-used tracking on keys (useful for finding dead keys; adds a write per verification)
- Scopes / permissions on keys
