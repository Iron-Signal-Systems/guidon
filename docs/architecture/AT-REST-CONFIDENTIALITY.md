# At-Rest Confidentiality Architecture

## Purpose

Guidon must state clearly what protects repository data, control artifacts, recovery copies, and sensitive authentication material when storage media, snapshots, replicas, or host filesystems are accessed outside the normal running service boundary.

## Initial Phase 1 Repository-data stance

Guidon Repository Format v1 identifies immutable recovery objects by SHA-256 of their exact logical stored bytes.

For the initial FreeBSD/ZFS implementation, Guidon does **not** introduce application-level per-object encryption into the Repository Format v1 object-identity contract.

Initial Repository-data at-rest confidentiality is provided by the deployment/storage layer using FreeBSD/OpenZFS capabilities where required by deployment policy.

The preferred production/pilot direction is:

```text
ZFS native encryption enabled for datasets containing customer recovery data
separate protection for Journal/private-key datasets according to their authority boundary
keys managed outside ordinary repository object content
```

## Important distinction: Job/control artifacts

The decision to defer **application-level encryption of Repository recovery objects** does not mean all Guidon data relies only on filesystem encryption.

Guidon Jobs are privileged control artifacts and follow the stronger rule in `../contracts/JOB-STORAGE-AND-MFA-V1.md`:

> **A complete signed Guidon Job must not be durably stored in plaintext.**

Any durable Guidon-controlled Job queue/spool/database representation is application-encrypted before persistence, in addition to any underlying filesystem/ZFS encryption.

This distinction is intentional:

```text
Repository recovery object
    -> exact logical bytes define object identity
    -> application-level object encryption deferred
    -> storage-layer encryption provides initial at-rest confidentiality

Guidon Job/control artifact
    -> signed exact Job bytes remain the authorization object
    -> encrypted envelope is the durable storage representation
    -> component-specific Job-storage key boundary
```

The encrypted Job envelope must decrypt back to the exact signed Job bytes before Job signature/authorization validation.

## Why application-level Repository-object encryption is deferred

Application-level encryption of Repository objects changes several foundational concerns:

- whether object identity hashes plaintext or ciphertext;
- deduplication behavior;
- key rotation/re-encryption semantics;
- Repository import/recovery without the original Controller;
- key escrow and disaster recovery;
- replication behavior; and
- the ability to verify historical objects independently.

Guidon will not silently add those semantics before they are explicitly designed and recovery-tested.

This is a deliberate scope boundary, not a claim that storage-layer encryption solves every threat.

## Threat boundary

Guidon distinguishes these cases:

### Normal service access

Repository/Journal Jail filesystem visibility, service identities, transport security, authority separation, Job envelope encryption, and MFA-secret protection govern their respective runtime data classes.

### Offline theft of unencrypted Repository media

If an attacker obtains readable unencrypted Repository storage outside Guidon's running security boundary, Guidon does **not** claim confidentiality of customer recovery data merely because object hashes verify.

Response:

```text
OUTSIDE CLAIM unless deployment storage encryption is enabled and its keys remain protected
```

Application-encrypted Job artifacts may remain confidential within their own key boundary even when the underlying filesystem is readable; that claim is limited by the strength/availability of the Job-storage KEK and does not extend to unrelated plaintext Repository objects.

### Offline theft of ZFS-encrypted media without keys

When ZFS native encryption is correctly enabled and the required key material is not available to the attacker, Guidon relies on OpenZFS encryption for the defined storage-at-rest confidentiality property.

Guidon records/configures that protection where it can establish it, but does not relabel a configured setting as proof that keys were never compromised.

### FreeBSD host-root compromise while keys are loaded

A malicious FreeBSD host root may potentially access decrypted datasets, process memory, mounted data, service configuration, and locally accessible keys.

ZFS encryption at rest and application-encrypted Jobs are not claimed defenses against a fully compromised running host that can inspect plaintext process memory or loaded decryption material.

Response:

```text
OUTSIDE CLAIM for confidentiality against complete malicious host root while protected data/keys are available to the host
```

### Windows SYSTEM/kernel compromise while Job keys are usable

A protected endpoint may durably spool Jobs only in encrypted form, but a sufficiently privileged running Windows compromise may inspect service memory, manipulate the service, or access locally available key-protection material.

Response:

```text
OUTSIDE CLAIM for confidentiality against complete malicious local OS authority while plaintext/key material is available
```

This does not weaken the value of preventing plaintext Job leakage through offline disks, ordinary application files, logs, snapshots, or support bundles.

### Repository Jail compromise

A Repository Jail compromise may expose Repository data that the Repository service is legitimately able to read. Dataset encryption does not prevent a compromised authorized reader from reading already-mounted data.

Authority separation still limits unrelated signing/authorization capabilities according to the Jail and identity contracts.

## Key separation

Storage-encryption key material is separate from:

```text
Journal Ed25519 attestation key
Controller Job signing key
AD Authorization Broker signing key
Recovery Authority key
endpoint/infrastructure transport keys
component-specific Job-storage KEKs
TOTP/MFA secret-protection keys
Recovery Copy administrator private keys
```

Likewise, Job-storage KEKs and MFA-secret protection keys are not reused as Repository/ZFS encryption keys.

A convenience design that stores all of these secrets in one Repository configuration file is prohibited.

## TOTP/MFA secret protection

TOTP shared secrets are authentication secrets, not ordinary configuration values.

They must be encrypted at rest under the verifier's own security boundary as defined in `JOB-STORAGE-AND-MFA-V1.md`.

The recovery/break-glass TOTP trust material must remain usable independently of production AD/Controller availability and must not be stored as plaintext merely because the system is considered an appliance.

## Repository portability

Repository recovery/import procedures must include the storage-encryption key/recovery requirement when encryption is enabled.

Guidon must never advertise an encrypted repository as self-recoverable from repository bytes alone if the required decryption key material is unavailable.

The truthful recovery state distinguishes:

```text
repository bytes present
storage encryption detected
required key available / unavailable / not_determined
repository readable / not_readable
```

Job-encryption KEK recovery is a different concern from Repository-object readability and is reported separately where queued Jobs must survive a component rebuild.

## Recovery Copy at-rest boundary

The Recovery Copy appliance defined in `RECOVERY-COPY.md` establishes its **own** at-rest protection and key hierarchy.

It must not assume that because the primary Repository uses encrypted ZFS datasets, a copied artifact is automatically protected by equivalent storage encryption on the independent system.

Intended separation:

```text
Primary Repository storage-encryption key
    != Recovery Copy storage-encryption key
    != Recovery Copy administrator RSA-4096 private key
    != Recovery Copy Job-storage KEK
```

A primary-appliance key compromise must not intentionally become the decryption key for the independent Recovery Copy storage domain.

## Integrity remains separate from confidentiality

ZFS encryption and authentication do not replace Guidon exact-byte SHA-256 object/manifest/record verification.

AES-GCM authentication of an encrypted Job envelope does not replace the Job's detached signature, authorization assertion, target/scope/expiry/replay checks, or Journal gating.

Likewise, SHA-256 integrity does not provide confidentiality.

Guidon reports these as different properties.

## Snapshots, replicas, exports, and recovery packages

Any future snapshot, replication, migration, removable-media, Recovery Copy, or off-site copy workflow must explicitly preserve or re-establish the intended at-rest confidentiality boundary.

Guidon must not assume that because the source dataset was encrypted, every exported copy remains equivalently protected.

Recovery Export package encryption/portable-media handling is frozen when the Recovery Copy export format is implemented. Until then Guidon makes no unsupported claim that an exported package remains confidential merely because the Recovery Copy appliance itself used encrypted storage.

## Phase 1 acceptance

Before a deployment is presented as having encrypted Repository storage, Guidon/testing should establish at minimum:

- dataset encryption state actually observed;
- intended key-load/unload behavior;
- reboot/import behavior with key unavailable;
- successful authorized recovery after key availability is restored;
- no Guidon private keys embedded inside repository object content; and
- documented operational recovery procedure for the storage-encryption key.

In addition, Phase 1 must establish for every durable Job/control artifact that Phase 1 actually persists:

- no complete plaintext Job body is present in the durable queue/spool/database;
- wrong Job-storage key cannot decrypt the artifact;
- modified ciphertext/tag fails authenticated decryption;
- decryption reproduces the exact signed Job bytes;
- logs/support artifacts do not contain plaintext Job bodies or OTP values; and
- the implementation's swap/pagefile/plaintext-memory boundary is documented and tested honestly.

Application-level Repository-object encryption remains deferred until an explicit later contract defines its identity, recovery, rotation, and portability semantics.
