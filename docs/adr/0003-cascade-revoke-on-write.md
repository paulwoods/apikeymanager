# Cascade-revoke keys when a parent is soft-deleted

Status: accepted

Soft-deleting an Application revokes all of its keys and soft-deletes its KeyOwners in the
same transaction; soft-deleting a KeyOwner revokes its keys likewise. The alternative — 
leaving keys `ACTIVE` and having verification join up to KeyOwner and Application to check
neither is deleted — was rejected.

Checking the chain at read time would put two JOINs and a wider predicate on the highest
volume operation in the system, reintroducing exactly the fail-open risk that
[ADR-0002](./0002-terminal-revocation-instead-of-soft-delete-on-keys.md) removed. Paying
the cost once at delete time keeps verification a single-row indexed lookup with no joins.

## Consequences

**Restoring a soft-deleted Application or KeyOwner does not restore its keys.** Because
revocation is terminal, un-deleting a parent recovers the record and its history, but every
key it held stays dead and must be reissued. This is intended: a tenant that was switched
off should not silently regain live credentials.

Deleting an Application with many keys is a bulk write rather than a single-row update, and
must be transactional — a failure part-way through has to roll back entirely, or the
Application survives with some of its keys already revoked.
