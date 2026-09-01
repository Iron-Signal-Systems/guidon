# Record v1 Contract

## Purpose

Record v1 is Guidon's immutable authoritative factual history.

Records are distinct from debug/application logs. They preserve operationally meaningful observations, actions, attempts, failures, refusals, state transitions, security changes, verifications, reconciliations, and other facts needed to explain what Guidon actually did or encountered.

## Core rules

- One meaningful fact/transition per record.
- Records are immutable after creation.
- State changes produce new records; old records are not edited.
- Failures, rejections, unavailable dependencies, and unknown states are recordable facts.
- Corrections do not rewrite old records; a later correction/reliability record references the earlier record.
- Current state may be derived/cached for operational use but never replaces the authoritative record history.
- Loss of the mutable catalog must not destroy the authoritative history.
- A state transition that requires a durable record is not complete until the required Record v1 bytes cross the defined durability boundary.
- A conclusion-like verification result must preserve the actual factual basis: method, expected value, observed value, and comparison/result.

## Authoritative finalization

Authoritative Record v1 bytes are finalized according to `AUTHORITATIVE-RECORD-FINALIZATION-V1.md`.

For normal Guidon operational records, the Repository authoritative-record boundary adds the required ISS-system clock observation and other Repository-side authoritative provenance **before** the final byte sequence is frozen, hashed, durably stored, and submitted to the Journal.

Producer-created observations or source artifacts are not silently treated as already-finalized authoritative Record v1 bytes. Where exact producer bytes must be preserved, they remain a separately identified source artifact referenced by the authoritative record.

After finalization, the Record v1 byte sequence is immutable.

## Identity and integrity

Every record has:

```text
schema = guidon.record
schema_version = 1
record_id = UUIDv7
record_type = controlled explicit type
```

The exact finalized Record v1 bytes are SHA-256 hashed outside the record by the Repository/Journal integrity path.

Do **not** embed `record_sha256` inside the exact bytes being hashed; that creates a self-reference problem.

`record_id` is the semantic record identity. The externally calculated SHA-256 is the exact-byte integrity/attestation value.

Record v1 exact-byte encoding/parsing follows `EXACT-BYTE-ENCODING-V1.md`.

## Record type

Use explicit controlled names, not multiple synonyms for the same event.

Examples include:

```text
OBJECT_RECEIVED
OBJECT_STORED
OBJECT_INTEGRITY_CHECKED
OBJECT_PUBLISHED
RECOVERY_POINT_PREPARED
RECOVERY_POINT_COMMITTED
AUTHORIZATION_CHECKED
JOURNAL_GAP_DETECTED
REPOSITORY_RECONCILIATION_STARTED
REPOSITORY_RECONCILIATION_COMPLETED
```

`OBJECT_PUBLISHED` is deliberately distinct from `RECOVERY_POINT_COMMITTED`. Object publication places one verified durable object into the immutable SHA-256 namespace; `COMMITTED` is a recovery-point lifecycle state defined by the Recovery Point Commit v1 contract.

Exact vocabulary evolves deliberately through the contract; implementations should not invent near-duplicate names casually.

## Time

Record v1 preserves distinct time facts, including at least:

```text
occurred_at
recorded_at
```

and the time provenance defined in `TIME-AND-CONFIGURATION-PROVENANCE.md`.

The required Repository-side `iss_system_clock` observation is inserted before authoritative Record v1 finalization according to `AUTHORITATIVE-RECORD-FINALIZATION-V1.md`.

When Journal-attested, Journal receipt/entry time remains separate from producer/Repository time.

## Producer provenance

Producer provenance is mandatory and includes enough to identify the component/build/runtime that created the record, for example:

```text
component
version
build_id
instance_id UUIDv7
```

A restart/new runtime instance gets a new instance identity where the component-instance contract requires it.

## Correlation identifiers

Use explicit UUIDv7 correlation identifiers where relevant, including:

```text
operation_id
job_id
recovery_point_id
endpoint_id
authentication_id
connection_id
```

A record contains only relationships that are actually applicable/known.

## Attribution

Record v1 keeps identity classes separate:

```text
user
gmsa
service_identity
endpoint_identity
transport_identity
producer
```

A gMSA is not encoded as a user.

Scheduled/system work may truthfully use `user.presence = not_present` while preserving the actual executing gMSA/service identity.

AD-related user/gMSA facts may include SID, account/domain, authenticating DC, event correlation, and provenance as actually observed.

## Network observations

When relevant, factual network observations may include:

```text
layer2_source_mac_observed
layer2_destination_mac_observed
observation_point
interface
vlan_id_if_observed
source/destination IP and port when observed
```

Record v1 does not automatically reinterpret a MAC as a gateway/router/server/direct-destination role.

## Subject model

Each record has one primary typed `subject` rather than a generic bag of unrelated identifiers.

`subject.type` selects the type-specific identity schema.

Examples:

### Repository object

```text
subject.type = repository_object
object_id = sha256:<...>
```

### Recovery point

```text
subject.type = recovery_point
recovery_point_id = UUIDv7
```

### Endpoint

```text
subject.type = endpoint
endpoint_id = UUIDv7
```

Hostname/machine SID remain observed attributes unless the particular record is specifically about one of those values.

### Windows file

Use stable native identifiers when actually observed, along with explicit requested/resolved path observations. A path is a locator, not assumed permanent object identity.

### Windows directory / volume

Directories use an explicit directory subject type. Volumes preserve observed volume GUID/serial identifiers where available; drive letter is an observed locator rather than permanent identity.

### Certificate

Exact certificate identity uses the SHA-256 of exact DER certificate bytes, plus serial/thumbprint/chain facts as applicable.

### AD user / gMSA

When the record is about the identity itself, SID is the key native identifier where observed.

### PostgreSQL / MSSQL

Use native workload identifiers where actually observed. Examples include PostgreSQL system identifier, database OID, relation OID, or future SQL Server native identifiers. Guidon does not invent uniqueness when the workload did not expose it.

### Configuration/job/operation/authentication

Use their defined UUIDv7 identities when they are the record subject.

## Relationships

Related entities belong in explicit relationships rather than overloading the primary subject.

For example, an object-verification record may have the object as primary subject and reference the verification epoch/recovery points through relationships.

## Result

Keep the common `result` small and precise. Type-specific factual data belongs in the record's typed `data` payload.

Avoid generic `SUCCESS`/`FAILURE` when a more exact state/result exists.

Examples of better factual results:

```text
match
mismatch
authorized
denied
not_determined
not_performed
connection_rejected
storage_write_failed
```

## Explicit semantic absence/unknown states

Use explicit semantic states instead of null/missing ambiguity where the distinction matters:

```text
not_present
not_known
not_observed
not_applicable
not_available
not_determined
not_performed
not_captured
```

These terms are not interchangeable.

## Observation versus interpretation

Authoritative Guidon records should avoid interpretive conclusions.

Accepted factual provenance categories may include concepts such as:

```text
observed
observed_local
authenticated
reported
resolved
cryptographically_verified
```

Any mechanically derived comparison must state its basis explicitly. Guidon does not use a derived value to turn incomplete topology/security facts into a conclusion for the administrator.

## Verification records

A verification record must identify:

```text
subject/target
method
expected value or condition
observed value or condition
result
verification time/context
```

Avoid an unqualified:

```text
verified: true
```

when the administrator cannot tell what was verified or how.

## Reconciliation records

Restart/reconciliation observations are new facts at the time they are made.

If a post-restart inspection finds a durable commit marker but the exact historical commit transition record is absent, Guidon may record that committed state was observed after restart and that the original transition time is `not_known`. It must not manufacture a pre-crash timestamp or event.

## Journal relationship

Every authoritative Record v1 is expected to become Journal-attested.

The Journal receives the exact finalized Record v1 bytes, independently calculates SHA-256, assigns Journal ordering, and returns a signed receipt. Synchronous versus asynchronous attestation behavior is defined by the Journal architecture contract.

Journal signature generation/verification follows `SIGNATURE-V1.md` for the applicable Journal purpose.
