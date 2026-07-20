# API Key Manager

Issues API keys to named subjects within a customer application, and answers whether a
presented key is currently valid. It is both the issuer and the verifier of every key it
manages.

## Language

### Tenancy

**Application**:
A customer system that consumes the API. The unit of tenancy — every KeyOwner and ApiKey
belongs to exactly one.
_Avoid_: Client, tenant, project, service

**KeyOwner**:
The subject an ApiKey is issued to, scoped to one Application. A data record only — a
KeyOwner never authenticates and never signs in.
_Avoid_: User, account, member

**Operator**:
A human who signs in to the management UI to issue and revoke keys. Authenticated
externally; identified in audit fields by an opaque string. Not stored as an entity.
_Avoid_: User, admin, actor

### Keys

**ApiKey**:
A credential issued to a KeyOwner, presented by a caller to authenticate an API request.
_Avoid_: Token, secret, credential

**Key Digest**:
The stored SHA-256 hash of an ApiKey. The only form of the key that this system retains.
_Avoid_: token_string, key hash, secret

**Key Prefix**:
A short leading fragment of an ApiKey, stored in the clear so operators can recognise a
key in the UI without it being usable.
_Avoid_: Display key, masked key, hint

**Issuance**:
Creating an ApiKey. The one and only moment its plaintext exists; it is shown to the
Operator once and is thereafter unrecoverable.
_Avoid_: Generation, minting, creation

**Revocation**:
Permanently ending an ApiKey's validity. Terminal — a revoked key is never restored;
access is restored by issuing a new one.
_Avoid_: Disable, deactivate, delete, soft-delete

**Expiry**:
An ApiKey passing its `expires_at` instant. Always derived by comparing that instant to
the current clock — never stored as a flag.
_Avoid_: Expired status, is_expired, lapsed

**Verification**:
Answering whether a presented key is currently valid: issued, not revoked, and not
expired.
_Avoid_: Validation, authentication, key check

**Verifier**:
A service that calls this system to verify a key it has been presented with. Holds its own
machine identity; never an Operator and never a KeyOwner.
_Avoid_: Consumer, caller, client

### Records

**Soft Delete**:
Marking an Application or KeyOwner as withdrawn from use while retaining its history.
Applies to those two only — an ApiKey is revoked, never soft-deleted.
_Avoid_: Archive, deactivate, disable
