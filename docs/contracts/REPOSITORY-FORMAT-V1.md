# Repository Format v1

## Purpose

Guidon recovery data must remain interpretable when the original Controller, catalog, database, host, or ZFS dataset layout is unavailable. This contract defines the minimum self-describing bootstrap information and verification-key portability required for Repository Format v1.

## Governing rule

> **Intact Guidon recovery data must be discoverable and interpretable without the original mutable catalog.**

The repository format may use ZFS for storage, but recovery meaning does not depend on one ZFS dataset name, pool name, controller database, or historic application host.

## Repository format descriptor

Every initialized Guidon repository contains one durable, immutable bootstrap descriptor for its format generation.

Conceptually:

```text
schema = guidon.repository-format
schema_version = 1
repository_id = UUIDv7
repository_format_version = 1
created_at
object_hash_algorithm = sha256
manifest_integrity_algorithm = sha256
record_integrity_algorithm = sha256
```

The descriptor uses Exact-Byte Encoding v1.

A repository implementation may store additional non-secret format metadata, but the descriptor cannot depend on the catalog to be found or interpreted.

## Stable discovery

The v1 implementation must provide one documented deterministic bootstrap location or discovery rule for the format descriptor.

The exact filesystem path is an implementation choice frozen before Phase 1 release, but it must remain discoverable from the repository root without database lookup.

A reasonable v1 direction is conceptually:

```text
/repository-format-v1.json
```

The implementation must not scan arbitrary customer object bytes and guess that a repository exists.

## Repository identity

`repository_id` is a stable UUIDv7 identity for one initialized Guidon repository authority.

Moving/mounting/importing the same intact repository does not create a new repository identity merely because:

- pool name changes;
- host changes;
- mount point changes;
- Jail name changes; or
- Controller/catalog is rebuilt.

A deliberately new repository initialization receives a new identity.

## Format compatibility

Repository software must inspect the format descriptor before modifying repository state.

Behavior is fail closed:

```text
supported exact format
    -> may proceed according to implementation compatibility rules

newer/unknown format
    -> read/write operations that could mutate state are refused

malformed descriptor
    -> integrity/format condition surfaced; do not initialize over it
```

Guidon must never overwrite an unknown repository with a fresh v1 descriptor merely because the catalog is absent.

## Minimum portable authority

Repository Format v1 preserves enough immutable/durable information to reconstruct supported recovery meaning from:

```text
format descriptor
immutable objects
immutable manifests
recovery-point publication artifacts
Repository authoritative Record v1 history
Journal receipts required by committed recovery points
Journal public verification-key history needed to verify those receipts
```

Derived catalog/index data is not part of the minimum authority.

## Journal verification-key portability

Journal private signing keys remain inside the Journal signing boundary and are not copied into the Repository.

Journal public verification material is not secret and MUST be preserved with Repository recovery metadata whenever a committed recovery point depends on a Journal signature created by that key generation.

The Repository retains, for every referenced Journal `signing_key_id`, the exact verification material needed for offline/historical verification.

For Ed25519 v1 this includes at minimum:

```text
signing_key_id
exact raw 32-byte public key or a lossless encoded representation
algorithm = ed25519
first_observed/accepted generation facts
key-transition/authorization references where applicable
```

The Repository independently recalculates:

```text
signing_key_id = sha256:<SHA-256 of exact raw public-key bytes>
```

and refuses a mismatching key file/material.

## Public-key history is not trust policy by itself

Possessing a public key proves only that signatures can be mathematically checked.

Historical trust/authorization for that key generation must also be established from Guidon's durable trust-transition/configuration history.

The Repository therefore does not treat an arbitrary public key dropped into the verification-key directory as an authorized historic Journal key.

## Recovery after Journal loss

If Journal service/storage is unavailable but Repository recovery data and required historical Journal public verification material survive, Guidon may verify historical Journal receipts/checkpoints within the surviving trust history.

It must accurately report the distinction between:

```text
historical signature verifies
current Journal service unavailable
current Journal continuity not established
```

Historical verification capability does not manufacture a replacement Journal signing authority.

## Layout independence

The physical layout beneath the repository root may evolve if the v1 format reader can deterministically discover the authoritative artifacts required by the version contract.

Object filenames/directories remain an implementation namespace for immutable SHA-256 objects; path alone is never proof of object integrity.

## Bootstrap integrity

The repository format descriptor is exact-byte SHA-256 protected as part of Repository authority. A future detached signature may be used according to Signature v1, but Format v1 does not require possession of an online repository-format signing private key for normal operation.

The implementation must define how the descriptor's expected hash is durably anchored within the repository bootstrap metadata without creating a circular dependency.

At minimum, startup/import performs an explicit integrity observation and records the exact bytes/hash it found.

## Import/rebuild behavior

A future Guidon installation rebuilding around intact repository data performs, conceptually:

```text
locate repository root
    -> read/validate format descriptor
    -> identify repository_id/format
    -> enumerate immutable recovery-point publication artifacts
    -> verify manifests
    -> verify referenced objects
    -> load/validate required Journal public verification-key history
    -> verify required Journal receipts/signatures
    -> rebuild derived catalog/index
    -> record all mismatches/gaps/unavailable conditions
```

The catalog is rebuilt from authority; authority is not rewritten to match the catalog.

## Acceptance tests

Before Phase 8 is considered complete, test at minimum:

- original Controller/catalog destroyed;
- Repository mounted at a different path/host;
- ZFS pool renamed/imported;
- Journal service unavailable but required historical public keys retained;
- unknown/newer repository format;
- malformed format descriptor;
- missing Journal public key for an otherwise present receipt;
- public-key bytes whose calculated ID does not match stored `signing_key_id`; and
- catalog rebuilt solely from intact Repository Format v1 authority.
