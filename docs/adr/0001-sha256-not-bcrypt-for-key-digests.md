# SHA-256, not bcrypt, for API key digests

Status: accepted

API keys are stored as a plain SHA-256 digest with a unique index, not as a bcrypt or
Argon2 hash. Deliberately-slow hashing exists to defend low-entropy secrets that humans
choose; our keys are 32 bytes from `SecureRandom`, so there is nothing to brute-force and
slow hashing buys no security.

## Considered Options

**bcrypt / Argon2** was rejected on two independent grounds. It defends against offline
guessing, which is already infeasible at 256 bits of entropy. More decisively, its per-row
salt makes it impossible to compute a digest from a presented key and look it up — every
verification would have to load every key row and run a deliberately-slow hash against
each. That is a full table scan on the hottest path in the system, and it does not scale
at any table size.

**HMAC with a server-side pepper** was rejected as defending a door with no lock to pick.
It adds a boot-time secret dependency and per-digest version tracking for rotation, in
exchange for hardening against an attack that entropy already prevents.

## Consequences

Verification is a single indexed lookup on the digest. Digest comparison is still done in
constant time, and the endpoint returns identical shape and timing for absent, revoked and
expired keys, so that neither existence nor state leaks.

If key entropy is ever reduced — shorter keys, a human-chosen component, a non-CSPRNG
source — this reasoning collapses entirely and the decision must be revisited.
