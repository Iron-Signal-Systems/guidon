# Repository Format v1

## Purpose

Guidon recovery data must remain interpretable when the original Controller, catalog, database, host, or ZFS dataset layout is unavailable. This contract defines the self-describing bootstrap, minimum authoritative layout, and Journal verification-key portability required for Repository Format v1.

## Governing rule

> **Intact Guidon recovery data must be discoverable and interpretable without the original mutable catalog.**

The repository format may use ZFS for storage, but recovery meaning does not depend on one ZFS dataset name, pool name, controller database, or historic application host.

## Repository root

Repository Format v1 has one logical repository root. Mount point, ZFS pool name, dataset name, host, and Jail name are deployment details and are not repository identity.

Within the logical repository root, these v1 bootstrap/authority locations are frozen:

```text
/guidon-repository.json
/guidon-repository.sha256
/objects/sha256/<first2>/<next2>/<full_digest>
/manifests/<recovery_point_id>/manifest.json
/manifests/<recovery_point_id>/manifest.sha256
/recovery-points/<recovery_point_id>/commit.json
/records/<implementation-bounded-layout>
/journal-receipts/<implementation-bounded-layout>
/journal-verification-keys/<signing_key_id>.pub
/journal-verification-keys/<signing_key_id>.json
```

`records` and `journal-receipts` may use bounded subdirectories/segments for scale, but their reader must be deterministic from Repository Format v1 metadata and may not require the catalog to discover authoritative content.

## Repository format descriptor

`/guidon-repository.json` is an immutable Exact-Byte Encoding v1 document containing at least:

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

`/guidon-repository.sha256` contains exactly:

```text
sha256:<64 lowercase hexadecimal characters>\n
```

where the digest is SHA-256 over the exact bytes of `/guidon-repository.json`.

The companion digest detects descriptor corruption/change but is not by itself a cryptographic trust anchor against an attacker able to replace both files. Historical Repository/Journal records and configured trust policy provide the applicable provenance/trust context.

Guidon must not claim otherwise.

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

Repository software inspects and verifies the format descriptor before modifying repository state.

```text
supported exact format
    -> may proceed according to v1 rules

newer/unknown format
    -> mutation refused

malformed descriptor or digest mismatch
    -> integrity/format condition surfaced; mutation refused
```

Guidon must never initialize over unknown or damaged repository bytes merely because the catalog is absent.

## Immutable object layout

Objects are exact immutable bytes identified by:

```text
sha256:<lowercase hex>
```

The final object path is:

```text
/objects/sha256/<first2>/<next2>/<full_digest>
```

where `<first2>` and `<next2>` are the first four lowercase hexadecimal digest characters split into two two-character directories and `<full_digest>` is the complete 64-character lowercase SHA-256 digest without the `sha256:` prefix.

Path identity never substitutes for hashing the exact object bytes.

Temporary incoming objects are outside the final immutable namespace and must be placed on a filesystem/dataset boundary that supports the Repository publication contract.

## Manifest layout

For each recovery point:

```text
/manifests/<recovery_point_id>/manifest.json
/manifests/<recovery_point_id>/manifest.sha256
```

`manifest.json` is the exact immutable versioned manifest.

`manifest.sha256` contains exactly:

```text
sha256:<64 lowercase hexadecimal characters>\n
```

for the exact manifest bytes.

## Recovery-point publication artifact

For each committed recovery point:

```text
/recovery-points/<recovery_point_id>/commit.json
```

`commit.json` is the immutable publication artifact defined by `RECOVERY-POINT-COMMIT-V1.md` and binds at least:

```text
schema/schema_version
recovery_point_id
manifest_sha256
preparation_record_id
preparation_record_sha256
Journal receipt reference/identity
published_at
repository_instance_id
```

Existence of the directory alone is not commitment. `commit.json` must parse and verify against all required authoritative artifacts.

## Minimum portable authority

Repository Format v1 preserves enough information to reconstruct supported recovery meaning from:

```text
format descriptor + digest
immutable objects
immutable manifests + digests
recovery-point commit artifacts
Repository authoritative Record v1 history
Journal receipts required by committed recovery points
Journal public verification-key history needed to verify those receipts
```

Derived catalog/index data is not part of the minimum authority.

## Journal verification-key portability

Journal private signing keys remain inside the Journal signing boundary and are never copied into the Repository by this contract.

For every Journal `signing_key_id` referenced by a committed recovery point, the Repository retains the exact public verification material and metadata required by the signature profile that created the referenced artifact.

The stable Repository Format v1 locations remain:

```text
/journal-verification-keys/<signing_key_id>.pub
/journal-verification-keys/<signing_key_id>.json
```

`.pub` contains the exact canonical public verification bytes defined by the applicable signature profile. `.json` identifies at least the `signing_key_id`, signature/profile version, algorithm, public-key encoding, first-observed/accepted generation facts, and trust/key-transition references where applicable.

For Ed25519 Signature v1, the existing representation is unchanged:

- `.pub` contains exactly the raw 32-byte Ed25519 public key;
- `.json` identifies `algorithm = ed25519` and the applicable Signature v1 profile; and
- `signing_key_id = sha256:<lowercase SHA-256 of exact raw 32-byte public key>`.

A future signature profile defines its own canonical public-key encoding and key-ID derivation rules. A future profile may remain within Repository Format v1 only when its verification material can be deterministically represented and discovered under this existing portability model without changing the meaning of existing v1 artifacts. Otherwise a new Repository Format version is required.

Cryptographic agility never authorizes an old reader to guess the algorithm or key encoding of unknown verification material. Unknown/unsupported signature profiles are surfaced and mutation/recovery claims fail closed according to compatibility policy.

## Public-key history is not trust policy by itself

Possessing a public key permits mathematical signature verification. It does not by itself prove that the key was an authorized Journal authority at the relevant time.

Historical authorization/trust must also be established from durable trust-transition/configuration history.

An arbitrary public key copied into `journal-verification-keys` does not become trusted merely because its self-derived key ID is internally consistent.

## Recovery after Journal loss

If Journal service/storage is unavailable but Repository recovery data and required historical public verification material survive, Guidon may verify historical Journal signatures within surviving trust history.

It reports separately:

```text
historical signature verification result
current Journal service availability
current Journal continuity/checkpoint state
```

Historical verification capability does not manufacture a replacement Journal signing authority.

## Import/rebuild behavior

A future Guidon installation rebuilding around intact Repository Format v1 data performs, conceptually:

```text
locate logical repository root
    -> verify guidon-repository.json + companion digest
    -> identify repository_id/format
    -> enumerate recovery-point commit artifacts
    -> verify manifests/digests
    -> verify referenced objects by SHA-256
    -> load/validate referenced Journal public verification material
    -> verify required Journal receipts/signatures
    -> verify authoritative Record history as defined
    -> rebuild derived catalog/index
    -> record mismatches/gaps/unavailable conditions
```

Authority is never rewritten to agree with a derived catalog.

## At-rest encryption interaction

Repository Format v1 object identity is SHA-256 of the exact logical recovery bytes stored by Guidon before any storage-layer encryption performed transparently by ZFS.

Storage-layer encryption does not change object IDs, manifests, or repository-format identities because encryption/decryption occurs below the Guidon file/object format boundary.

Application-level per-object encryption is not part of Repository Format v1.

## Acceptance tests

Test at minimum:

- original Controller/catalog destroyed;
- repository mounted at a different host/mount point;
- ZFS pool renamed/imported;
- Journal unavailable but required historical public keys retained;
- unknown/newer repository format;
- malformed descriptor;
- descriptor digest mismatch;
- missing Journal public key for a referenced receipt;
- public-key bytes whose calculated ID mismatches metadata/path;
- unknown/unsupported Journal signature profile or public-key encoding is surfaced and never autodetected/reinterpreted as Ed25519;
- missing/invalid recovery-point `commit.json`; and
- catalog rebuilt solely from intact Repository Format v1 authority.
