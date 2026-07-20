# Terminal revocation instead of soft-delete on ApiKey

Status: accepted

Everything else in this system is soft-deleted; `ApiKey` is not. A key has a single
`status` of `ACTIVE` or `REVOKED` plus an `expires_at`, revocation is permanent, and there
is no `deleted_at` column and no un-revoke.

The original design called for `active` + `expiration_date` + `deleted_at` on keys. That is
three independent signals meaning the same thing — "this key must not authenticate" —
producing eight combinations, most of them meaningless. It also made verification a
multi-clause predicate, and verification is the one query in the system that **fails open**
when a clause is forgotten.

## Considered Options

**Reversible revocation** was rejected because "re-enable the key we thought was leaked" is
not a decision anyone should be able to make by misclicking. Access is restored by issuing
a new key.

**Keeping soft-delete on keys for uniformity** was rejected because nothing in the product
can produce that state. The UI offers issue and revoke; there is no distinct meaning for
"deleted" that "revoked" does not already carry.

## Consequences

Verification is exactly `status = 'ACTIVE' AND expires_at > now()`.

Expiry is never stored as a flag. It is derived by comparing `expires_at` to the clock, so
a key expiring requires no scheduled job and no state change — the row simply stops
satisfying the predicate. Materialising it into an `is_expired` column would mean the
database is lying between job runs.
