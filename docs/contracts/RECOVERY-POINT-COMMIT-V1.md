# Recovery Point Commit v1

## Purpose

This contract freezes the minimum durable artifacts and Journal binding required before Guidon may report a recovery point as `COMMITTED`.

## Governing rule

> **A recovery point is committed only when the exact recovery definition, its required data, its preparation record, its Journal attestation, and its durable publication state are all bound to one recovery-point identity.**

## Commit identity

Every recovery point has:

```text
recovery_point_id = UUIDv7
manifest_sha256 = SHA-256(exact immutable manifest bytes)
```

The recovery-point identity is independent of object/content identity.

## Required preparation record

Before publication, the Repository creates one immutable authoritative Record v1 representing recovery-point preparation/commit intent.

That record MUST bind at least:

```text
recovery_point_id
manifest_sha256
manifest schema
manifest schema_version
capture operation_id
source/endpoint identity
repository configuration generation identity/hash
verification policy generation identity/hash if applicable
```

The exact Record v1 bytes are finalized under `AUTHORITATIVE-RECORD-FINALIZATION-V1.md` and hashed as exact bytes.

The Journal must attest those exact preparation-record bytes before recovery-point publication crosses the commit boundary.

## Required object and manifest facts

Before the preparation record may authorize commit, the Repository must establish:

1. every object referenced as required by the manifest exists in the immutable object namespace;
2. every required object crossed the defined durability boundary;
3. every required object's exact bytes verify against its SHA-256 identity;
4. the exact manifest bytes crossed the defined durability boundary;
5. the exact manifest SHA-256 matches `manifest_sha256`; and
6. all required manifest references are structurally and semantically valid for the supported schema.

A catalog row or mutable cache cannot satisfy these requirements by itself.

## Journal binding

The Journal receipt used to gate commit must bind the exact preparation Record v1 through:

```text
record_id
record_sha256
journal_stream_id
stream_sequence
journal_entry_sha256
signing_key_id
```

The preparation Record v1 itself binds `recovery_point_id` and `manifest_sha256`.

Therefore the Journal attestation transitively binds the exact immutable recovery definition without requiring the Journal to parse customer workload/recovery semantics.

The Journal remains a witness, not the owner of recovery meaning.

## Repository publication artifact

After the required signed Journal receipt is durably stored at the Repository, the Repository may durably publish the recovery point.

Repository Format v1 must preserve a recovery-point publication artifact/marker that binds at least:

```text
schema/schema_version
recovery_point_id
manifest_sha256
preparation_record_id
preparation_record_sha256
journal_receipt_id or exact receipt reference
journal_receipt_sha256 if the receipt is separately hashed by repository format
published_at
repository_instance_id
```

The publication artifact itself is immutable once committed.

## Commit sequence

Conceptually:

```text
verify/durably store required objects
    -> durably store exact manifest
    -> verify manifest + references
    -> finalize/durably store preparation Record v1
    -> submit exact preparation record to Journal
    -> receive/verify signed Journal receipt
    -> durably store receipt
    -> durably publish immutable recovery-point marker
    -> only now report COMMITTED
```

## No false commit

Guidon must never report `COMMITTED` when any required step above is absent, failed, not determined, or not durably established.

Examples:

```text
Journal unavailable
    -> recovery point may remain prepared
    -> COMMITTED = not_performed

manifest references missing object
    -> COMMITTED = not_performed

receipt signature invalid
    -> COMMITTED = not_performed

publication marker write/durability failure
    -> COMMITTED = not_performed
```

## No silent non-commit

A blocked commit is itself meaningful operational state and produces factual records identifying the exact blocking condition.

Later successful continuation does not erase the earlier blocked/failure history.

## Restart reconciliation

After restart, the Repository reconstructs commit state from durable artifacts, not catalog claims.

Examples:

```text
manifest valid + preparation record valid + receipt absent
    -> prepared; Journal attestation may be resumed if facts remain valid

receipt valid + publication artifact absent
    -> not previously committed merely because publication was likely next
    -> Repository may publish now if every current requirement remains valid

publication artifact valid + catalog entry absent
    -> committed Repository authority may rebuild catalog projection

catalog says committed + publication artifact absent/invalid
    -> mismatch; catalog cannot create commit truth
```

Guidon never backdates `published_at` or invents a missing historical commit transition.

## Immutability

Changing any of the following creates a different recovery definition and therefore cannot mutate an already committed recovery point:

```text
manifest bytes
manifest_sha256
recovery_point_id
required object references
source identity
capture operation relationship
```

A correction/new capture creates a new recovery point or new immutable historical record as appropriate.

## Acceptance tests

At minimum test interruption/crash immediately before and after each durability boundary, including:

- after last object durability but before manifest durability;
- after manifest durability but before preparation record finalization;
- after preparation record durability but before Journal submission;
- after Journal durable acceptance but before receipt reaches Repository;
- after receipt durability but before publication;
- during publication durability; and
- after publication but before catalog update.

Every observed restart state must resolve to one deterministic truthful result or require operator review when facts conflict.
