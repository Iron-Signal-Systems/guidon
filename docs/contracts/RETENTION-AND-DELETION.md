# Retention and Deletion Contract

## Principle

Destroying recovery data is fundamentally different from creating it.

Guidon never physically removes committed recovery data merely because it appears old or unreferenced. Deletion requires a defined authority path, immutable historical records, independent Journal attestation at the required boundary, and a fresh verification that the specific data is eligible for removal.

The governing rules are:

> **No silent deletion. No silent non-deletion.**

> **When Guidon cannot establish that deletion is safe, the default action is to retain the data and surface why deletion was not performed.**

## Recovery-point deletion and object deletion are separate

Normal administrators/retention policies act on recovery points, not raw SHA-256 object paths.

Conceptually:

```text
DELETE_RECOVERY_POINT
    recovery_point_id = UUIDv7
```

Objects may be shared by many recovery points, so removal of one recovery point does not imply its referenced objects may be deleted.

The lifecycle is:

```text
recovery-point deletion
    -> reference reconciliation
    -> object garbage-collection candidate identification
    -> independent object-reference verification
    -> physical object deletion if authorized/safe
```

## Manual deletion

Manual recovery-point deletion uses Job v1 and the applicable current authorization path.

Because deletion destroys recovery capability, it is a high-impact operation and is Journal-gated before physical removal begins.

There is no generic `force`/`delete_anyway` escape hatch that bypasses the defined safety and authorization contract.

## Scheduled retention

Scheduled retention is distinct from a human directly performing deletion.

A scheduled retention execution truthfully records:

```text
user.presence = not_present
```

and references the exact immutable retention policy/version that authorized evaluation.

The administrator who created a policy months earlier is not falsely represented as the person physically deleting a recovery point at the scheduled execution time.

## Versioned policy history

Retention policies are immutable/versioned.

A change from one retention rule to another creates a new policy generation. Historical deletion/evaluation records preserve the exact policy ID/version/SHA-256 that was used.

Older policy generations remain explainable after later changes.

## Eligibility is not deletion

Guidon distinguishes:

```text
eligible_for_deletion

deletion_authorized

deletion_started

deletion_completed
```

A policy evaluation that identifies a recovery point as eligible does not itself mean any bytes were removed.

## Journal-gated destructive boundary

Before committed recovery-point data is physically removed:

1. the deletion request/evaluation facts are durable;
2. required authorization is established;
3. the deletion-authorization record is durable;
4. the Journal durably attests the required record;
5. the signed receipt is durable at the Repository; and
6. only then may physical deletion begin.

If Journal attestation is unavailable, deletion is `not_performed` and the exact blocking condition is recorded.

Guidon does not make destructive retention "best effort" by journaling it later.

## Fresh verification before destructive action

Immediately before physical deletion, Guidon re-establishes the facts needed for the action from authoritative Repository state rather than trusting a stale catalog decision.

For recovery-point deletion this includes, as applicable:

- target recovery-point UUID exists;
- commit marker is present/valid;
- exact manifest SHA-256 verifies;
- target identity/manifest identity match the authorized request;
- the governing policy/authorization is the intended immutable version;
- current eligibility is established under that policy; and
- required Journal receipt/signature is valid.

The catalog may identify candidates but does not authorize deletion.

## Historical identity retained after bytes are removed

If recovery-point data is legitimately removed, authoritative records retain enough history to establish that the recovery point existed and why/when it was removed.

At minimum retain facts such as:

```text
recovery_point_id
manifest_sha256
capture time/workload identity as defined by Record v1
endpoint/source identity
policy/authority used
deletion operation/job identity
deletion time/result
```

Whether the full manifest itself is retained after content deletion is a later metadata-retention policy decision; historical Record v1/Journal facts are not rewritten to pretend the recovery point never existed.

## Object garbage collection

Objects referenced by a deleted recovery point become garbage-collection **candidates**, not immediate deletion targets.

A catalog/reference count may accelerate candidate discovery, but:

> **A database reference count of zero is not sufficient authority to physically delete an object.**

Before deletion, Guidon must establish that no retained committed recovery-point manifest references the object within the authoritative reference scope.

A reverse index/reference count may be maintained for performance, but the destructive decision must be rooted in committed manifest authority rather than a mutable SQL count alone.

## Object GC records

Meaningful GC activity produces explicit records, for example:

```text
OBJECT_GC_CANDIDATE_IDENTIFIED
OBJECT_REFERENCE_CHECK_STARTED
OBJECT_REFERENCE_CHECK_COMPLETED
OBJECT_DELETION_AUTHORIZED
OBJECT_DELETION_STARTED
OBJECT_DELETED
```

A reference check records the actual scope and observed result, such as:

```text
object_id
reference_check_scope
retained_references_observed
result
```

Guidon does not turn `no reference observed` into a stronger conclusion than the scope/check actually established.

## Concurrency/race prevention

The implementation must provide a synchronization boundary preventing this race:

```text
GC:        object verified unreferenced
Commit:    new recovery-point manifest durably references object
GC:        deletes object
```

Object reference creation and physical GC eligibility must be coordinated so a committed recovery point cannot acquire a reference that the deletion path failed to see.

The exact locking/index/transaction implementation remains open, but the invariant is fixed.

## Post-failure reverification blocks automatic destruction

If an unclean shutdown, power loss, storage fault, or uncertain shutdown has triggered a verification epoch and the retained Repository has not completed the required post-event verification, automatic object GC is blocked.

Automatic retention deletion is also blocked while the Repository's integrity/reference state is uncertain.

This is deliberately conservative: a storage event is not the time to start destroying old recovery data before Guidon has re-established what actually survived.

## Integrity mismatch blocks automatic deletion

An object or recovery-point integrity mismatch does not make the affected bytes an automatic garbage-collection target.

Guidon records/surfaces the mismatch and blocks automatic deletion of the affected material so potentially important recovery/investigation data is not destroyed simply because it is damaged.

Any later removal follows an explicit defined authority path.

## Interrupted deletion

A crash/power loss during recovery-point or object deletion is reconciled from observed durable state after restart.

Guidon does not fabricate a historical `DELETION_COMPLETED` event merely because a path is now absent.

It records the post-restart observations, for example:

```text
RECOVERY_POINT_DELETION_STATE_OBSERVED_AFTER_RESTART
OBJECT_ABSENCE_OBSERVED_AFTER_RESTART
```

and continues only where durable facts identify one deterministic safe action.

Conflicting/ambiguous states require administrator review.

## Ransomware/control-plane separation

A normal protected endpoint backup identity does not inherently possess recovery-point deletion authority.

A compromised Controller alone must not silently become unquestioned repository-wipe authority. Destructive actions still traverse the defined Job/authorization/Repository-validation/Journal-gating boundaries.

This reduces blast radius but does not claim that compromise of every required authority can be defeated.

## Break-glass separation

Recovery Authority credentials authorize the defined recovery scope. They do not automatically authorize repository wipe, retention-policy changes, Journal history deletion, or trust-anchor changes.

Emergency destructive administration, if ever needed, must be a separate explicitly defined authority/policy.

## Historical records outlive content retention

When recovery bytes legitimately age out, Guidon Record v1 and Journal history may still show:

```text
recovery point created
objects/manifests committed
verification performed
restore/validation operations performed
delete eligibility established
deletion authorized
deletion physically performed
```

Physical content retention and authoritative historical record retention are separate concerns.
