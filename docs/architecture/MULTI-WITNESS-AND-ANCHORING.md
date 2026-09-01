# Multi-Witness and External Anchoring Architecture

## Purpose

Guidon's initial implementation uses one operational Journal. The mature design allows multiple independent Journal witnesses plus an external write-once advancing historical anchor without turning the Repository into an active-active distributed system.

This document defines the intended trust roles and the invariants Phase 1 must preserve.

## Governing principle

> **Redundancy must not collapse independent trust domains into one shared identity, key, stream, or mutable state.**

A second Journal is not a byte-for-byte mirror of the first Journal Jail. It is a separate witness with its own identity, signing key, durable stream history, sequence numbers, and checkpoints.

The external witness is not a third operational Journal. It is an isolated historical anchor whose application-visible state is write-once and always advancing.

## Intended mature topology

```text
SYSTEM 1 — Guidon Recovery Appliance
    Repository
    Journal A

SYSTEM 2 — Guidon Witness Appliance
    Journal B

SYSTEM 3 — External Witness
    isolated host/appliance
    COW / append-oriented storage
    write once at application/protocol level
    always advancing anchor head
```

The systems occupy separate physical/admin failure domains where deployment requires that protection.

A second Jail on the same FreeBSD host can improve process/Jail availability but is not counted as an independent host failure domain.

## Journal A and Journal B

Each operational Journal has separate:

```text
journal_instance_id
Journal service identity
transport identity
Ed25519 attestation key
signing_key_id
Journal streams
stream sequence numbers
segments
checkpoints
local durable storage
```

No Journal private key is cloned merely to make a redundant instance.

Each Journal independently receives the same exact authoritative Record v1 bytes from the Repository and independently calculates:

```text
record_id
record_sha256
```

They may assign different:

```text
journal_stream_id
stream_sequence
journal_entry_id
segment_id
checkpoint_id
journaled_at
```

That is expected and correct.

## Cross-Journal correlation

Journal witnesses correlate the same authoritative fact using:

```text
record_id
+
exact record_sha256
```

They do **not** require identical internal streams or segment boundaries.

A future Guidon status/reconciliation process may compare witnesses:

```text
same record_id + same record_sha256
    -> consistent independent witness observations

same record_id + different record_sha256
    -> JOURNAL_WITNESS_CONFLICT
```

A witness conflict is a serious integrity/security condition and must not be reduced to ordinary replication lag.

## Operational gating policy

Phase 1 retains the existing one-Journal requirement.

A future two-Journal deployment may define a normal availability policy such as:

```text
configured witnesses = 2
normal required witnesses = 1
healthy desired witnesses = 2
```

This means:

```text
2 of 2 available
    -> normal operation, full witness redundancy

1 of 2 available
    -> normal policy may permit gated operation
    -> Guidon reports degraded witness redundancy

0 of 2 available
    -> required Journal attestation unavailable
    -> gated operation fails closed
```

This is an availability policy, not a claim that 1-of-2 provides the same compromise resistance as 2-of-2.

For especially sensitive operations, later policy may require stronger witness participation, for example:

```text
Journal witness-set change
Journal authority transition
trust-root transition
external-witness trust transition
```

Such requirements must be explicit per operation rather than implied by the existence of multiple Journals.

## Degraded state must be visible

Guidon should expose witness health separately from operation eligibility.

Example:

```text
Operational Journal witnesses configured: 2
Healthy: 1
Required for this operation: 1

Journal A: unavailable
Journal B: healthy / attestation durable

Operation Journal gate: satisfied
Witness redundancy: degraded
```

Guidon must not display a generic green `Journal OK` when configured redundancy has been lost.

## External witness role

The external witness has a different mission:

> **Preserve an independently controlled historical high-water mark showing that specific sealed Journal checkpoint states already existed.**

It does not:

- own backup/recovery semantics;
- receive customer recovery objects;
- perform restores;
- issue Jobs;
- authorize normal operations;
- sign Journal A/B receipts;
- normally participate in synchronous recovery-point commit gating; or
- provide generic remote administration of Guidon.

Normal backup availability therefore does not depend on the external witness being online.

## External witness inputs

The external witness receives only the minimum checkpoint-anchor material needed for its purpose, conceptually including:

```text
source_journal_id
journal_checkpoint_id
journal_checkpoint_sha256
journal_signing_key_id
source Journal checkpoint/signature material or verification reference
source first/last sequence and entry count where part of the anchor schema
```

It does not need customer file contents, database contents, Repository manifests, user file paths, or normal Repository credentials.

## External witness anchor chain

Each accepted anchor receives a monotonically increasing witness sequence and commits to the prior durable witness anchor.

Conceptually:

```text
schema = guidon.external-witness-anchor
schema_version = 1
anchor_id = UUIDv7
witness_id
witness_sequence
previous_anchor_sha256
source_journal_id
journal_checkpoint_id
journal_checkpoint_sha256
journal_signing_key_id
received_at
witness_instance_id
```

For the first anchor:

```text
previous_anchor_sha256 = not_applicable
```

For every later anchor:

```text
witness_sequence = previous durable sequence + 1
previous_anchor_sha256 = SHA-256(exact previous anchor bytes)
```

The external witness may sign its own acknowledgement/anchor artifact using a key separate from Journal A/B keys.

## Write-once, always-advancing behavior

At the application/protocol level the external witness provides no normal operation equivalent to:

```text
DeleteAnchor
EditAnchor
SetSequence
SetHead
Rollback
Truncate
ReplaceHistory
SignArbitraryBytes
Execute
```

Accepted anchor history is immutable. The durable head only advances.

A duplicate submission of the same source checkpoint may be handled idempotently; conflicting bytes/identity for the same claimed checkpoint produce an integrity conflict rather than replacement.

## Copy-on-write storage

Copy-on-write/append-oriented storage is the intended storage behavior because it naturally avoids in-place application updates to prior anchor data.

However:

```text
COW != WORM
```

A privileged host administrator may still be able to delete files/snapshots, alter storage metadata, destroy the filesystem, restore an older disk image, replace software, or access local keys.

Guidon therefore does not claim that COW alone protects witness history from malicious host root.

## Rollback threat

A hash chain can prove internal continuity of the history presented to the verifier. By itself it cannot prove that the entire witness storage was not rolled back to an older previously valid state.

Example:

```text
legitimate previous head = 18493
restored old snapshot head = 17102
```

The old state may be internally valid but stale.

The mature design should therefore provide an independently protected high-water-mark/rollback-detection mechanism where the deployment needs that claim. Possible later implementations include hardware-backed monotonic state, external anchoring, or another independently controlled persistence point.

Until such a mechanism is implemented and tested, malicious witness-host root capable of rolling back the entire witness state is `OUTSIDE CLAIM`.

## Root compromise boundaries

Guidon states these boundaries plainly:

```text
Repository Jail compromise
    != automatic Journal signing-key compromise

System 1 host root compromise
    may compromise Repository + Journal A on that host
    != automatic Journal B compromise
    != automatic external-witness compromise

System 2 host root compromise
    may compromise Journal B on that host
    != automatic System 1 compromise
    != automatic external-witness compromise

External witness host root compromise
    may compromise the external witness within locally accessible authority
    != automatic Repository/Journal A/Journal B compromise

all relevant hosts/keys/authorities compromised
    -> outside Guidon's protection claim
```

The objective is compartmentalization and independent contradiction points, not an impossible claim that software survives total privileged compromise of every trust domain.

## External witness availability behavior

External anchoring is asynchronous relative to ordinary Repository operation unless a future explicit policy says otherwise.

Guidon should surface:

```text
last anchored checkpoint
last anchored time
oldest unanchored checkpoint age/count
external witness reachable/unreachable
external witness verification status
```

An external-witness outage should normally produce an `anchoring delayed/degraded` state, not stop ordinary backup ingestion or normal Journal-gated commit.

## Phase 1 requirements

Phase 1 implements only one Journal, but must preserve these future invariants:

- Record v1 identity/hash is stable and sufficient for cross-Journal correlation;
- Journal identity and signing-key identity are explicit;
- Journal stream sequence is local to one Journal and never treated as global Guidon order;
- checkpoint artifacts are independently hashable/signable;
- Repository commit artifacts reference the specific Journal receipt(s) that satisfied the current policy rather than assuming one timeless Journal implementation forever; and
- APIs/data structures do not require future Journal B to clone Journal A internal state.

Phase 1 does **not** implement distributed consensus, Journal quorum, external anchoring, or witness failover.

## Later acceptance tests

When multi-witness support is implemented, test at minimum:

- Journal A unavailable / Journal B healthy under 1-of-2 normal policy;
- Journal B unavailable / Journal A healthy;
- both unavailable;
- different Journal internal sequence numbers for the same Record v1;
- same `record_id`/same hash across witnesses;
- same `record_id`/different hash conflict;
- witness signing-key rotation independently on A/B;
- external witness offline while operational Journals continue;
- external witness resumes and anchors backlog in order;
- attempted anchor deletion/edit/head reset rejected at application API;
- old external-witness storage snapshot rollback detected when high-water-mark protection is implemented; and
- compromise/loss of one witness does not silently relabel the surviving protection state as fully redundant.
