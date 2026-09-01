# At-Rest Confidentiality Architecture

## Purpose

Guidon must state clearly what protects repository data when storage media, snapshots, replicas, or host filesystems are accessed outside the normal running service boundary.

## Initial Phase 1 stance

Guidon Repository Format v1 identifies immutable recovery objects by SHA-256 of their exact logical stored bytes.

For the initial FreeBSD/ZFS implementation, Guidon does **not** introduce application-level per-object encryption into the Repository Format v1 object-identity contract.

Initial at-rest confidentiality is provided by the deployment/storage layer using FreeBSD/OpenZFS capabilities where required by deployment policy.

The preferred production/pilot direction is:

```text
ZFS native encryption enabled for datasets containing customer recovery data
separate protection for Journal/private-key datasets according to their authority boundary
keys managed outside ordinary repository object content
```

## Why application-level object encryption is deferred

Application-level encryption changes several foundational concerns:

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

Repository/Journal Jail filesystem visibility, service identities, transport security, and authority separation govern normal runtime access.

### Offline theft of unencrypted media

If an attacker obtains readable unencrypted Repository storage outside Guidon's running security boundary, Guidon does **not** claim confidentiality of customer recovery data.

Response:

```text
OUTSIDE CLAIM unless deployment storage encryption is enabled and its keys remain protected
```

### Offline theft of ZFS-encrypted media without keys

When ZFS native encryption is correctly enabled and the required key material is not available to the attacker, Guidon relies on OpenZFS encryption for the defined storage-at-rest confidentiality property.

Guidon records/configures that protection where it can establish it, but does not relabel a configured setting as proof that keys were never compromised.

### FreeBSD host-root compromise while keys are loaded

A malicious FreeBSD host root may potentially access decrypted datasets, process memory, mounted data, service configuration, and locally accessible keys.

ZFS encryption at rest is not a claimed defense against a fully compromised running host with decryption keys loaded.

Response:

```text
OUTSIDE CLAIM for confidentiality against complete malicious host root while data is available to the host
```

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
```

A convenience design that stores all of these secrets in one Repository configuration file is prohibited.

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

## Integrity remains separate from confidentiality

ZFS encryption and authentication do not replace Guidon exact-byte SHA-256 object/manfiest/record verification.

Likewise, SHA-256 integrity does not provide confidentiality.

Guidon reports these as different properties.

## Snapshots, replicas, and exports

Any future snapshot, replication, migration, removable-media, or off-site copy workflow must explicitly preserve or re-establish the intended at-rest confidentiality boundary.

Guidon must not assume that because the source dataset was encrypted, every exported copy remains equivalently protected.

## Phase 1 acceptance

Before a deployment is presented as having encrypted Repository storage, Guidon/testing should establish at minimum:

- dataset encryption state actually observed;
- intended key-load/unload behavior;
- reboot/import behavior with key unavailable;
- successful authorized recovery after key availability is restored;
- no Guidon private keys embedded inside repository object content; and
- documented operational recovery procedure for the storage-encryption key.

Application-level object encryption remains deferred until an explicit later contract defines its identity, recovery, rotation, and portability semantics.
