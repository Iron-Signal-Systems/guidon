# Journal Architecture

## Purpose

The Guidon Journal is an independent witness for authoritative Record v1 history.

The Repository owns backup/recovery semantics. The Journal owns authenticated record submission, independent exact-byte hashing, durable append/order, hash continuity, receipts/checkpoints, and its signing authority.

The Repository does not possess the Journal signing private key. The Journal does not own customer recovery logic or browse customer backup content.

## Submission contract

For normal Guidon operational history, the Repository submits the **exact finalized authoritative Record v1 bytes** created under `AUTHORITATIVE-RECORD-FINALIZATION-V1.md`.

Other Guidon components may produce source observations/event material for Repository finalization, but they do not bypass the authoritative Repository finalization boundary merely because they can reach the Journal.

Journal-internal system records are the exception: they are finalized within the Journal's own authoritative system-record boundary and remain in the dedicated Journal system stream.

The Journal independently:

1. authenticates the authorized Repository producer identity or the defined Journal-internal producer boundary;
2. validates the common Record v1 envelope needed for Journal operation using the strict exact-byte parser profile;
3. calculates SHA-256 over the exact submitted bytes;
4. checks Record ID idempotency/conflict rules;
5. assigns Journal stream/sequence/entry identity;
6. durably appends the entry;
7. produces a signed receipt after the required Journal durability boundary; and
8. returns that receipt to the Repository for externally submitted records.

The Journal never trusts a producer-supplied record hash without independently calculating it.

## Idempotency

For a previously seen `record_id`:

```text
same record_id + same exact-byte SHA-256
    -> idempotent; return existing durable receipt

same record_id + different exact-byte SHA-256
    -> reject; record integrity/security conflict
```

The Journal does not silently replace the old bytes.

## Streams and ordering

Journal ordering is claimed only where it can be proven.

The initial model uses one Journal stream per authenticated producer instance/lifetime. Stream identity is UUIDv7.

Within one stream:

```text
stream_sequence = 1, 2, 3, ...
```

Every entry after the first references the exact-byte SHA-256 of the previous Journal entry.

Conceptually:

```text
journal_entry_id
journal_stream_id
stream_sequence
record_id
record_sha256
previous_entry_sha256
received_at
producer_instance_id
```

For sequence 1, `previous_entry_sha256` is `not_applicable`.

A restarted producer receives a new producer-instance identity and normally a new Journal stream rather than pretending the new runtime instance is the previous one.

Cross-stream relationships use explicit IDs such as:

```text
operation_id
job_id
connection_id
authentication_id
recovery_point_id
endpoint_id
```

Guidon does not invent one total global order across independent streams from wall-clock timestamps alone.

## Segments

Streams use bounded immutable segments rather than one unbounded mutable file.

A segment closes on mechanical boundaries such as:

- maximum entry count;
- maximum byte size;
- maximum age;
- controlled shutdown; or
- Journal signing-key rotation.

The exact tuning values remain implementation details. Low-volume streams must still eventually seal by age or lifecycle boundary.

Once sealed, a segment is immutable.

## Checkpoints

A sealed segment produces an exact-byte checkpoint with fields conceptually including:

```text
schema = guidon.journal-checkpoint
schema_version
checkpoint_id
journal_stream_id
segment_id
first_sequence
last_sequence
entry_count
first_entry_sha256
last_entry_sha256
segment_sha256
previous_checkpoint_sha256
created_at
signing_key_id
```

The first checkpoint uses `previous_checkpoint_sha256 = not_applicable`.

The exact checkpoint bytes are signed using Signature v1 with:

```text
purpose = GUIDON:JOURNAL-CHECKPOINT:V1
algorithm = ed25519
```

The previous checkpoint's exact-byte SHA-256 links sealed segments within the stream.

## Receipts

A per-record receipt proves that the Journal durably accepted the record and assigned it a place in a stream.

Conceptually:

```text
receipt_id
record_id
record_sha256
journal_stream_id
stream_sequence
journal_entry_sha256
journaled_at
journal_instance_id
signing_key_id
```

The exact receipt bytes are signed using Signature v1 with:

```text
purpose = GUIDON:JOURNAL-RECEIPT:V1
algorithm = ed25519
```

A receipt does not claim a segment checkpoint already exists. An entry may be durable in an active segment while the covering checkpoint is `not_yet_created`.

Receipts and checkpoints have different purposes:

```text
receipt
    per-record durable Journal acceptance

checkpoint
    sealed stream/segment continuity attestation
```

## Journal durability

The Journal checkpoint/stream position never advances beyond entries that are durably recoverable.

The conceptual path is:

```text
receive exact record
    -> independently hash
    -> create Journal entry
    -> append
    -> synchronous durability boundary
    -> advance durable stream state
    -> sign required receipt/checkpoint material using Signature v1
    -> durably preserve required signing/output state
    -> return receipt
```

The implementation must explicitly test the exact filesystem/syscall boundary on the target FreeBSD/OpenZFS version.

## Crash behavior

After restart the Journal verifies the exact durable entry/segment/checkpoint state it can observe.

If an active segment had durable entries but no checkpoint before the crash, the new instance may create a post-restart checkpoint only after verifying those entries. It does not backdate or claim the checkpoint existed before the crash.

If a final entry is torn/incomplete, verified continuity ends at the last entry whose bytes, sequence, and hash relationship verify. The Journal records the incomplete state rather than silently pretending it never existed.

Journal gaps, sequence conflicts, hash-chain failures, and checkpoint mismatches are explicit records. Reconciliation never manufactures missing entries to make continuity look clean.

## Journal signing

Record/object integrity uses SHA-256. Journal receipt/checkpoint/key-transition attestation uses Ed25519 under `SIGNATURE-V1.md`.

For Ed25519:

```text
signing_key_id = sha256:<lowercase SHA-256 of exact raw 32-byte public key>
```

Public verification keys must be retained with recoverable Journal history for as long as the records need verification.

In addition, `REPOSITORY-FORMAT-V1.md` requires the Repository to retain the exact public verification material needed to verify every Journal key generation referenced by a committed recovery point. The Journal private key is never copied into the Repository by that rule.

The Journal private key remains inside the Journal signing boundary. The normal architecture does not expose a generic signing oracle. Signing operations are constrained to defined Journal semantics such as:

```text
SignJournalReceipt
SignJournalCheckpoint
SignJournalKeyTransition
```

The Journal transport mTLS key is separate from the Journal Ed25519 attestation key.

## Key rotation

Normal signing-key rotation:

1. stops/seals active segments under the old key;
2. creates final old-key checkpoints;
3. records/attests a transition between key generations using `GUIDON:JOURNAL-KEY-TRANSITION:V1`;
4. cryptographically links old/new generations, ideally with a transition attested by both where available; and
5. begins new segments/checkpoints under the new key.

Lost key is not the same as normal rotation. If the old private key is unavailable, Guidon does not manufacture cryptographic continuity. Historical artifacts continue to verify with the old public key; the new generation explicitly records the absence of old-key signing continuity.

The normal pre-alpha design does not automatically back the Journal private key into the Repository. Future HSM, escrow, external signer, or checkpoint anchoring may be added if justified.

## Journal's own records

The Journal records its own meaningful behavior, including as applicable:

```text
JOURNAL_STARTED
JOURNAL_STOPPED
stream created/closed
receipt issued
checkpoint created/signed
signing key loaded/rotated
submission rejected
record ID conflict
hash mismatch
chain failure
gap observed
storage low/exhausted
write/durability failure
```

Journal internal records use a dedicated Journal system stream and the same integrity/segment/checkpoint principles. The Journal does not need to send those records through its own network API or through the Repository merely to manufacture Repository provenance.

External callers cannot choose arbitrary Journal system streams or set their own sequence numbers.

## Narrow API

The Journal API is intentionally narrow. It must not provide general operations such as:

```text
Delete
Edit
SetSequence
SignAnything
Execute
```

## Synchronous gating versus durable asynchronous attestation

Every normal authoritative Record v1 is expected to become Journal-attested. The distinction is whether the associated action may proceed before the attestation exists.

### `GATED`

Journal attestation and the returned receipt must be durable before the associated high-impact action/state transition begins.

Initial gated boundaries include:

- recovery-point publication/commit;
- restore/recovery authorization before target modification;
- replacement of an existing recovery target;
- recovery-point deletion;
- physical object deletion/garbage collection;
- retention deletion;
- trust-anchor changes;
- endpoint certificate binding/authorization changes;
- Controller/Auth Broker/Journal signing authority transitions;
- break-glass authorization;
- temporary recovery-administrator authorization;
- recovery bootstrap activation; and
- other future destructive or privilege-escalating operations.

If Journal attestation is unavailable, the gated action is `not_performed` and the blocking condition is durably recorded locally for later attestation.

### `DURABLE_ASYNC`

Facts that have already occurred or high-volume observations are durably recorded locally first and may be Journal-attested afterward.

Examples include:

- object receive/store observations;
- object/manifest verification results;
- connection and network observations;
- certificate observations;
- user/gMSA authentication observations;
- AD query observations;
- transfer progress;
- storage observations;
- post-failure reverification results;
- operation completion/failure after the event has already happened; and
- crash/restart reconciliation observations.

The local pending queue cannot exist only in RAM. After restart, missing valid Journal receipts are discoverable from durable authoritative record state and resubmitted idempotently.

Guidon never discards older unjournaled records to make room for newer ones. If local authoritative-record durability can no longer be provided, new operations stop or stop at a defined safe boundary rather than silently losing history.

## Availability behavior

A temporary Journal outage may allow normal backup ingest to continue through durable objects/manifests/authoritative records, but a recovery point cannot cross the Journal-gated commit boundary until the required attestation exists.

The Journal backlog must be operationally visible, including at least the number of pending attestations, oldest pending time, and any gated operations blocked by Journal unavailability.
