# Repository Architecture

## Purpose

The Guidon Repository owns backup/recovery storage semantics. It stores exact immutable objects, immutable recovery-point manifests, recovery-point publication state, authoritative Record v1 history, and Journal receipts returned to the Repository.

The catalog is an acceleration/projection layer. It is not the sole authority for intact recovery data.

## Core data model

Guidon separates four concepts:

```text
OBJECT
    exact immutable bytes possessed by the Repository

MANIFEST
    immutable meaning/reconstruction description

RECOVERY POINT
    one specific capture event

RECORD
    immutable factual history of Guidon behavior and observations
```

### Object identity

The primary key for a Repository object is the SHA-256 digest of its exact bytes:

```text
sha256:<lowercase hex>
```

An object contains no filename, timestamps, ACL, owner, workload meaning, or version semantics. A changed byte creates a different object identity.

One file/database/volume may reference one object or many objects. Chunking strategy is deliberately not frozen by this contract.

A conceptual object namespace is:

```text
objects/sha256/<first2>/<next2>/<full_digest>
```

The exact filesystem layout may change if the identity and immutability contract remains intact.

### Recovery-point identity

A recovery point uses a UUIDv7 identity independent of content identity. Identical content captured on different days produces different recovery points.

### Manifest

A recovery point references one immutable, versioned JSON manifest. Guidon hashes the exact stored manifest bytes with SHA-256; integrity does not depend on JSON canonicalization.

Conceptually:

```text
manifest.json
manifest.sha256
```

A future detached `manifest.sig` may be added without changing the exact-byte integrity rule.

The generic envelope includes at least:

```text
schema
schema_version
recovery_point_id
created_at
source
workload.type
workload.schema_version
capture.started_at
capture.completed_at
capture.operation_id
content
```

Workload-specific content is typed rather than represented by one generic property bag.

Object references are explicit and ordered where ordering matters, with fields such as:

```text
object_id
logical_offset
length
```

Everything required for a supported recovery must be reconstructible from the immutable manifest plus referenced immutable objects. The original mutable catalog must not be required.

Manifests distinguish semantic states such as:

```text
not_present
not_captured
not_observed
not_known
not_applicable
```

Missing/empty fields must not silently carry multiple meanings.

## Object receive and publication

The required object path is conceptually:

```text
receive stream
    -> calculate SHA-256 while receiving
    -> write temporary file
    -> complete expected transfer
    -> finalize and verify hash
    -> synchronous durability boundary
    -> atomically publish into final SHA-256 namespace
```

An incomplete upload is never a committed object.

If an object with the same SHA-256 identity already exists, Guidon independently hashes/verifies the existing object before reusing it. A conflicting existing object is an integrity condition; Guidon does not silently overwrite it.

The temporary object must be placed so that final publication can use the defined atomic filesystem operation. The exact syscall sequence and directory-metadata durability behavior must be tested on the target FreeBSD/OpenZFS version rather than assumed.

## Durability

`write()` success is not durability.

`STORED` may be reported only after the exact bytes have crossed Guidon's defined synchronous durability boundary.

For the initial FreeBSD/ZFS implementation:

- Guidon durable datasets must not use `sync=disabled`;
- `sync=standard` is the minimum expected dataset behavior;
- the application explicitly uses synchronous durability operations at required boundaries;
- `sync=always` is not required merely to compensate for missing application durability calls;
- ZIL exists independently of a separate SLOG;
- a SLOG is a performance option, not a correctness requirement; and
- any production SLOG should use appropriate power-loss protection and redundancy.

ZFS checksums are useful storage protection but do not replace Guidon's SHA-256 object identity and verification.

### `STORED`

For an object, `STORED` means all of the following were actually performed:

- the complete expected bytes were received;
- SHA-256 was finalized;
- the required synchronous durability operation completed; and
- Guidon is willing to rely on those bytes surviving process/host restart without requiring the sender to recreate them.

### Object publication

Final object publication means the verified durable temporary object was atomically placed in the final immutable SHA-256 namespace and the required namespace/metadata durability boundary completed.

## Recovery-point commit contract

A recovery point does not become `COMMITTED` until every required condition is satisfied:

1. every object required by the manifest is durable;
2. every required object has the defined integrity verification result;
3. the exact manifest is durable;
4. the manifest SHA-256 verifies;
5. the required immutable preparation/commit-intent record is durable;
6. that required record is independently Journal-attested;
7. the signed Journal receipt is durable at the Repository; and
8. the Repository's durable commit publication/marker is completed.

Two failure rules are mandatory:

> **No false commit:** Guidon never reports `COMMITTED` before every required condition is satisfied.

> **No silent non-commit:** if a required condition blocks commit, Guidon immediately records and surfaces the exact blocking condition.

A later successful continuation does not erase the earlier failure/unavailability record.

The catalog cannot make a recovery point committed merely by saying it is committed. Repository-level durable artifacts are authoritative.

## Crash/restart reconciliation

After restart, Guidon reconstructs state only from durable artifacts it can actually observe and verify. It does not assume an operation completed because it had started before the crash.

Startup reconciliation begins with observation before mutation. Guidon examines, as applicable:

- temporary objects;
- final objects;
- manifests and manifest hashes;
- recovery-point commit markers;
- authoritative records;
- Journal receipts; and
- catalog/index state.

Meaningful reconciliation creates Record v1 history.

### Examples

**Temporary object, final object absent:** record the temporary object's presence, length, and calculated SHA-256. Continue only if durable facts identify one valid continuation path.

**Temporary and final object both present:** independently verify both. If both match the same expected object identity, deterministic cleanup may remove the duplicate temporary object after recording it. A mismatch requires operator visibility.

**Final object present:** hash exact bytes and compare with the SHA-256 encoded by the object identity. The filename/path alone is not proof.

**Manifest present, hash artifact absent:** Guidon may calculate a new post-restart observation of the manifest hash, but it must not pretend a historical `manifest.sha256` durability boundary occurred before the crash.

**Manifest/hash valid, Journal receipt absent:** the recovery point may remain prepared; Guidon can resume Journal attestation if the durable facts support that continuation.

**Journal receipt present, commit marker absent:** Guidon does not backdate commitment. If every current requirement remains valid, it may publish the recovery point now and record the actual current transition.

**Commit marker present, catalog missing/stale:** verify the commit artifacts and rebuild the derived catalog entry.

**Catalog says committed but Repository commit marker is absent/invalid:** report the mismatch; the catalog does not override Repository authority.

**Repository and Journal disagree on a record hash:** stop automatic reconciliation for the conflicting item and surface both observed hashes. Do not overwrite one to match the other.

**Interrupted restore:** inspect and record the target's current state; do not fabricate a prior `RESTORE_COMPLETED` event.

**Interrupted temporary recovery-account cleanup:** recovery is not finally validated until account removal and bootstrap disarm are actually verified.

## Deterministic continuation rule

Guidon may automatically continue reconciliation only when durable, independently verifiable facts identify one valid next action.

If facts conflict or permit multiple interpretations, Guidon records the discrepancy and requires administrator review.

Cleanup is never silent. The pattern is:

```text
observe
    -> identify
    -> verify
    -> record
    -> remove/continue if policy permits
    -> record the action
```

## Post-failure reverification

Any of the following starts a new repository verification epoch:

- unclean shutdown;
- host crash;
- power loss;
- shutdown state not determined;
- storage/pool/device integrity fault;
- meaningful I/O integrity error; or
- another event for which Guidon can no longer rely on the previous current-integrity state.

Each epoch has a UUIDv7 identity.

The old verification records remain true historical records, but no retained recovery point may inherit its **current** integrity status across the event.

Guidon re-verifies all retained recovery data, including old recovery points that may be needed months later during ransomware recovery.

At minimum the post-event pass verifies:

- every unique retained object by SHA-256;
- every retained manifest by exact-byte SHA-256;
- required object references from every retained committed recovery point;
- recovery-point publication artifacts;
- Journal receipts/signatures required by the recovery point;
- Journal continuity/checkpoint facts required by the defined verification; and
- catalog projections against Repository authority.

A shared object referenced by many recovery points needs one physical SHA-256 pass per verification epoch; the recovery points can reference that epoch's exact object-verification fact.

Integrity mismatches are recorded and surfaced immediately when observed. Guidon does not wait for the full repository scan to finish before reporting the first problem.

Historical `VALIDATED` restore records remain historical facts. A storage event does not mean the workload was re-restored/revalidated; it means the currently stored bytes require new integrity verification.

Automatic destructive garbage collection and automatic retention deletion are suspended while post-failure integrity/reference verification is incomplete.

## Storage layout and failure domain

A reasonable initial dataset direction is:

```text
guidon/repository/objects
guidon/repository/recovery-points
guidon/repository/records

guidon/journal/entries
guidon/journal/checkpoints
guidon/journal/keys
```

Separate datasets help enforce ownership, reservations, and operational behavior. Datasets on the same ZFS pool still share a physical failure domain; Guidon must not describe them as independent physical copies.

Storage-full, `ENOSPC`, synchronous write failure, or I/O error means the affected durability/commit state did not occur. Guidon records and surfaces the exact condition and stops any gated transition that requires the missing durability.

## Testing requirement

Process-kill tests are useful but are not sufficient to validate storage durability. The implementation must deliberately test crashes at each boundary and eventually include real abrupt power-loss testing on representative FreeBSD/ZFS hardware.
