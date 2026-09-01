# Recovery Point Commit v1

## Purpose

This contract freezes the minimum durable artifacts and Journal binding required before Guidon may report a recovery point as `COMMITTED`.

The v1 structure must work with the Phase 1 single-Journal deployment without hard-coding the Repository format to one Journal forever.

## Governing rule

> **A recovery point is committed only when the exact recovery definition, its required data, its preparation record, the Journal attestation policy in force, enough valid Journal receipts to satisfy that policy, and its durable publication state are all bound to one recovery-point identity.**

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
journal attestation policy generation identity/hash
```

The exact Record v1 bytes are finalized under `AUTHORITATIVE-RECORD-FINALIZATION-V1.md` and hashed as exact bytes.

The required operational Journal witness set must attest those exact preparation-record bytes sufficiently to satisfy the referenced Journal-attestation policy before recovery-point publication crosses the commit boundary.

## Required object and manifest facts

Before the preparation record may authorize commit, the Repository must establish:

1. every object referenced as required by the manifest exists in the immutable object namespace;
2. every required object crossed the defined durability boundary;
3. every required object's exact bytes verify against its SHA-256 identity;
4. the exact manifest bytes crossed the defined durability boundary;
5. the exact manifest SHA-256 matches `manifest_sha256`; and
6. all required manifest references are structurally and semantically valid for the supported schema.

A catalog row or mutable cache cannot satisfy these requirements by itself.

## Journal attestation policy

The Journal requirement is expressed as an immutable/versioned policy generation rather than an assumption that exactly one Journal exists forever.

Conceptually the policy establishes facts such as:

```text
policy_generation_id
policy_generation_sha256
configured authorized Journal identities
minimum required valid witnesses for this operation class
stronger witness requirement if applicable
```

Phase 1 uses the simple policy:

```text
configured operational Journals = 1
required valid Journal receipts for recovery-point commit = 1
```

A later two-Journal deployment may use a normal policy such as:

```text
configured operational Journals = 2
required valid Journal receipts = 1
healthy desired witnesses = 2
```

or a stronger policy where a specific high-risk operation requires both.

The policy that was actually in force is historical fact and is bound into the preparation/commit artifacts. A later policy change does not retroactively rewrite what satisfied an older commit.

## Journal receipt binding

Each Journal receipt used to satisfy the policy must bind the exact preparation Record v1 through:

```text
record_id
record_sha256
journal_instance_id / authorized Journal identity
journal_stream_id
stream_sequence
journal_entry_sha256
signing_key_id
```

The preparation Record v1 itself binds `recovery_point_id`, `manifest_sha256`, and the Journal-attestation policy generation.

Therefore each Journal attestation transitively binds the exact immutable recovery definition without requiring the Journal to parse customer workload/recovery semantics.

The Journal remains a witness, not the owner of recovery meaning.

## Receipt set

The Repository stores the complete set of Journal receipts it used to establish the commit gate.

Conceptually:

```text
journal_attestation:
    policy_generation_id
    policy_generation_sha256
    required_witness_count
    satisfied_witness_count
    receipts[]
```

Each `receipts[]` element identifies at least:

```text
journal_instance_id
journal_receipt_id
journal_receipt_sha256 if separately hashed by Repository Format
journal_signing_key_id
```

For Phase 1, `receipts[]` contains exactly one required receipt.

For future multi-Journal operation, receipt order in exact publication artifacts must be deterministic. The v1 direction is to sort receipt entries lexicographically by stable `journal_instance_id` before finalizing the publication artifact.

A second receipt received after the recovery point is already committed does not mutate the immutable original commit artifact. It is recorded as a later protection/attestation fact or through a separately defined immutable supplemental artifact.

## Repository publication artifact

After enough valid signed Journal receipts are durably stored at the Repository to satisfy the referenced policy, the Repository may durably publish the recovery point.

Repository Format v1 must preserve a recovery-point publication artifact/marker that binds at least:

```text
schema/schema_version
recovery_point_id
manifest_sha256
preparation_record_id
preparation_record_sha256
journal_attestation_policy_generation_id
journal_attestation_policy_generation_sha256
required_witness_count
satisfied_witness_count
journal_receipts[]
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
    -> resolve/freeze Journal-attestation policy generation
    -> finalize/durably store preparation Record v1
    -> submit exact preparation record to required/available authorized Journal witness(es)
    -> receive/verify signed Journal receipt(s)
    -> durably store receipts
    -> verify policy threshold satisfied
    -> durably publish immutable recovery-point marker
    -> only now report COMMITTED
```

Phase 1 performs this sequence with one Journal and one required receipt. The contract shape still remains witness-set compatible.

## No false commit

Guidon must never report `COMMITTED` when any required step above is absent, failed, not determined, or not durably established.

Examples:

```text
required Journal witness threshold unavailable
    -> recovery point may remain prepared
    -> COMMITTED = not_performed

manifest references missing object
    -> COMMITTED = not_performed

required receipt signature invalid
    -> COMMITTED = not_performed

valid receipts present but policy threshold not satisfied
    -> COMMITTED = not_performed

publication marker write/durability failure
    -> COMMITTED = not_performed
```

## Degraded witness redundancy versus commit eligibility

In a future multi-Journal policy, operation eligibility and redundancy health are separate facts.

Example:

```text
configured witnesses = 2
required for normal commit = 1
valid receipts obtained = 1

commit gate = satisfied
witness redundancy = degraded
```

The recovery point may become `COMMITTED` under that explicit policy, but Guidon must not label Journal witness redundancy fully healthy.

If policy requires two valid witnesses and only one receipt is available, commit remains blocked.

## Journal witness conflict

If different authorized Journals present the same authoritative:

```text
record_id
```

with different exact:

```text
record_sha256
```

Guidon treats that as a witness integrity/security conflict, not merely an insufficient receipt count.

The exact operation response is policy-defined, but Guidon must surface the conflict explicitly and must not silently choose whichever receipt allows a commit.

## No silent non-commit

A blocked commit is itself meaningful operational state and produces factual records identifying the exact blocking condition.

Later successful continuation does not erase the earlier blocked/failure history.

## Restart reconciliation

After restart, the Repository reconstructs commit state from durable artifacts, not catalog claims.

Examples:

```text
manifest valid + preparation record valid + insufficient valid receipts
    -> prepared; Journal attestation may be resumed if facts/policy remain valid

policy threshold satisfied + publication artifact absent
    -> not previously committed merely because publication was likely next
    -> Repository may publish now if every current/historical requirement remains valid

publication artifact valid + catalog entry absent
    -> committed Repository authority may rebuild catalog projection

catalog says committed + publication artifact absent/invalid
    -> mismatch; catalog cannot create commit truth
```

Guidon never backdates `published_at` or invents a missing historical commit transition.

A restart must evaluate the Journal policy generation referenced by the preparation/commit artifacts rather than silently applying a newly changed policy to rewrite historical commit truth.

## Immutability

Changing any of the following creates a different recovery definition/commit fact and therefore cannot mutate an already committed recovery point:

```text
manifest bytes
manifest_sha256
recovery_point_id
required object references
source identity
capture operation relationship
Journal-attestation policy generation used for commit
receipt set used in the immutable commit artifact
```

A correction/new capture creates a new recovery point or new immutable historical/supplemental record as appropriate.

## Acceptance tests

At minimum test interruption/crash immediately before and after each durability boundary, including:

- after last object durability but before manifest durability;
- after manifest durability but before preparation record finalization;
- after preparation record durability but before Journal submission;
- after Journal durable acceptance but before a receipt reaches Repository;
- after first receipt durability but before required policy threshold is satisfied where multi-witness testing applies;
- after required receipt threshold is satisfied but before publication;
- during publication durability; and
- after publication but before catalog update.

Phase 1 must also test that its one-Journal publication artifact uses the witness-set-compatible structure with `required_witness_count = 1`, not a special one-off schema that later requires Repository Format redesign.

Every observed restart state must resolve to one deterministic truthful result or require operator review when facts conflict.
