# Authoritative Record Finalization v1

## Purpose

This contract defines exactly where Guidon authoritative Record v1 bytes become final and how producer time facts and the ISS-system clock observation coexist without mutating an already-finalized record.

## Governing rule

> **The Repository authoritative-record boundary finalizes authoritative Record v1 bytes.**

An endpoint, Controller, Authorization Broker, Journal, recovery environment, or other Guidon component may produce source observations or proposed record material, but those bytes are not an authoritative Record v1 merely because they were produced by that component.

The authoritative Repository record path:

```text
receive producer observation/event material
    -> authenticate/identify producer as applicable
    -> preserve producer-supplied factual bytes/fields without rewriting their meaning
    -> add Repository-side ISS-system clock observation
    -> add authoritative producer/provenance references
    -> validate Record v1 schema and semantic requirements
    -> serialize one final Record v1 byte sequence
    -> freeze exact bytes
    -> calculate SHA-256 over those exact bytes
    -> cross the defined Repository record durability boundary
    -> submit those exact bytes to the Journal
```

After the `freeze exact bytes` boundary, the Record v1 bytes are immutable.

## Producer material is not silently rewritten

Producer-created observations remain attributable to the producer and retain their original observed values and times.

If a producer created material while disconnected and the Repository received it later, Guidon preserves both:

```text
producer occurred_at / recorded_at facts
Repository ISS-system observed_at
Journal received/journaled time
```

The Repository does not rewrite the producer event time to the Repository receipt time.

## ISS-system clock observation

The Repository adds the required `iss_system_clock` observation before authoritative Record v1 finalization.

At minimum it identifies:

```text
observed_at
system
instance_id
```

and includes synchronization/source facts when actually observed.

Because this observation is inserted before finalization, adding it does not mutate a previously hashed authoritative Record.

## Source-record preservation

When Guidon must preserve an exact producer-created byte artifact for independent integrity, replay, or protocol reasons, those bytes are stored/referenced as a separate typed source artifact or submitted-event artifact.

The authoritative Record v1 may reference that artifact by identity and exact-byte SHA-256.

A producer source artifact and an authoritative Record v1 are therefore distinct concepts:

```text
SOURCE ARTIFACT
    exact bytes produced elsewhere

AUTHORITATIVE RECORD V1
    final immutable factual history created at the Repository authoritative-record boundary
```

Guidon does not claim that Repository finalization makes producer-supplied content true; it records the provenance and facts it can establish.

## Journal relationship

The Journal receives the exact finalized authoritative Record v1 bytes.

The Journal independently calculates SHA-256 over those exact bytes. The Repository-calculated and Journal-calculated hashes must match for the same authoritative Record.

The Journal does not add or rewrite Record v1 fields before hashing/attestation.

## Repository-local records

Records whose producing component is the Repository still pass through the same finalization rule. The Repository producer facts and the Repository-side ISS-system clock observation may refer to the same host while remaining distinct fields with distinct meanings.

## Journal-local records

Journal internal system records are authoritative within the Journal system stream according to the Journal architecture. They are finalized within the Journal's own authoritative record boundary and use an identified Journal-system clock observation. They are not sent through the Repository merely to manufacture Repository provenance.

## Failure behavior

If Guidon cannot establish the required finalization facts or cannot durably preserve the final exact Record v1 bytes, the authoritative record transition did not complete.

Guidon must not:

- hash one byte sequence and later persist a different one under the same `record_id`;
- add fields after exact-byte hashing;
- silently normalize producer facts into different meanings; or
- manufacture an ISS-system observation for a time it did not actually observe.

## Implementation invariant

For one `record_id`:

```text
one authoritative finalized byte sequence
one externally calculated exact-byte SHA-256
zero mutations after finalization
```

A later correction is a new Record v1 that references the earlier record; it does not rewrite the earlier bytes.
