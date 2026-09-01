# Recovery Copy Appliance Architecture

## Purpose

The Recovery Copy appliance is a separate Guidon system whose purpose is to preserve an independently controlled recovery copy outside the primary Guidon appliance failure and administrative domain.

It is **not** a second active Repository and does not participate in normal Repository ownership, catalog mutation, or recovery execution.

## Governing principle

> **The primary Guidon appliance may push new immutable recovery material to the Recovery Copy appliance, but its normal replication identity does not receive read, modify, delete, or execute authority over the copy.**

Recovery access is an exceptional administrator-controlled path under a separate trust root.

## Intended topology

```text
SYSTEM 1 — Guidon Recovery Appliance
    Repository
    Journal A
        |
        | authorized push-only replication
        v
SYSTEM 4 — Recovery Copy Appliance
    independent recovery storage
    independent verification
    separate recovery administration/export interface
    separate recovery trust root
```

The Recovery Copy appliance should occupy a separate physical/admin failure domain when the deployment intends to claim protection from loss or compromise of the primary appliance.

## Not active-active

Guidon intentionally does not make the Recovery Copy appliance a writable peer Repository.

The design avoids:

- active-active Repository ownership;
- split brain;
- distributed locking;
- simultaneous recovery-point publication authority;
- mutable catalog synchronization as authority;
- quorum/consensus for normal Repository writes; and
- a second control plane capable of independently changing primary Repository state.

The primary Repository remains the normal authority for recovery-point publication.

## Replication ingress authority

The Recovery Copy appliance exposes a narrow replication-ingress interface that authenticates an explicitly authorized Guidon appliance identity.

The replication identity may perform only defined operations such as:

```text
SubmitImmutableObject
SubmitManifest
SubmitRecoveryPointPublicationArtifact
SubmitRequiredJournalReceipt
SubmitHistoricalJournalPublicVerificationMaterial
SubmitOtherExplicitPortableRecoveryMetadata
GetSubmissionReceipt / GetSubmittedArtifactStatus
```

The replication identity does not receive operations equivalent to:

```text
ReadObject
DownloadRecoveryPoint
BrowseCustomerRecoveryContent
DeleteObject
DeleteRecoveryPoint
ModifyObject
ReplaceManifest
SetRetention
Execute
RemoteShell
ExportRepository
```

If future replication status queries are provided, they reveal only the minimum metadata required to establish copy status; they do not become a recovery-content read API.

## Independent verification on receipt

The Recovery Copy appliance does not trust the primary's statement that content is correct.

For immutable objects it independently:

```text
receive bytes
    -> calculate SHA-256 itself
    -> compare with declared object identity
    -> write temporary storage
    -> cross its own durability boundary
    -> atomically publish immutable copy
    -> verify retained copy according to policy
```

It similarly validates supported manifests, recovery-point publication artifacts, Journal receipts/signatures, and historical verification-key material according to their contracts.

A malformed/conflicting artifact is rejected and recorded; an existing immutable artifact is never overwritten to make a conflict disappear.

## Replication status is separate from COMMITTED

Primary Repository recovery-point state remains:

```text
COMMITTED
```

when the normal Recovery Point Commit v1 requirements are satisfied in the authoritative primary Repository.

Recovery Copy state is a separate protection property, for example:

```text
recovery_copy:
    configured = true
    state = pending / receiving / stored / verified / failed / unavailable
    copy_appliance_id
    last_observed_at
```

Guidon must not redefine `COMMITTED` to mean that an independent copy exists unless a future explicit policy creates a separate service-level requirement.

This allows a locally usable recovery point to remain truthful during a WAN/copy outage while clearly exposing that independent-copy protection is incomplete.

## Recovery Copy receipt

After independently verifying a submitted recovery point/artifact set, the Recovery Copy appliance may issue a signed receipt/status artifact conceptually including:

```text
schema
schema_version
copy_receipt_id UUIDv7
copy_appliance_id
source_repository_id
recovery_point_id
manifest_sha256
objects_expected
objects_received
objects_verified
copy_state
completed_at
signing_key_id
```

The exact schema/signature purpose is frozen when Recovery Copy implementation begins.

A receipt proves only the explicitly stated Recovery Copy acceptance/verification facts; it does not claim the protected workload itself has been recovered or validated.

## Primary compromise resistance

A compromised primary Guidon appliance or stolen replication credential must not, by that fact alone, obtain normal authority to erase or retrieve the Recovery Copy contents.

This creates the intended asymmetry:

```text
Primary -> Recovery Copy
    submit new allowed immutable material

Primary -> Recovery Copy
    no ordinary recovery-content read
    no ordinary delete
    no ordinary modification
    no generic execution
```

This property must be enforced by the Recovery Copy's own API authorization and local storage/service permissions, not merely by UI convention.

## Recovery Copy retention/deletion

The Recovery Copy appliance owns its own retention/deletion authority for physical copies.

The primary may report facts such as:

```text
primary recovery point no longer locally retained
primary retention state changed
```

but those facts are not remote delete commands.

Physical destruction of Recovery Copy data requires Recovery Copy-local policy and the explicit authorization path defined for that appliance.

A future product may support minimum retention, delayed deletion, legal/administrative holds, or dual authorization, but the invariant is fixed now:

> **Normal primary replication credentials cannot directly erase the independent recovery copy.**

## Separate recovery administration trust root

Recovery/export administration uses a trust hierarchy separate from normal Guidon transport/endpoint/Repository/Journal PKI.

The intended direction is:

```text
Offline Recovery Copy Root CA
        |
        v
Recovery Copy Administration Intermediate CA
        |
        v
RSA-4096 Recovery Copy administrator certificate
```

The exact CA deployment/rotation profile is frozen before Recovery Copy implementation ships.

The root private key should normally remain offline. The administrative certificate is not an ordinary Guidon endpoint or Repository certificate.

Normal Guidon credentials that are intentionally **not** sufficient for Recovery Copy export include:

```text
protected endpoint certificate
Repository transport certificate
Journal transport certificate
normal Controller service identity
normal replication certificate
Controller Job signing key
AD Authorization Broker signing key
Journal signing key
```

## Recovery Copy MFA

Recovery Copy export administration requires the separate recovery identity plus policy-defined MFA under `../contracts/JOB-STORAGE-AND-MFA-V1.md`.

The RSA-4096 administrator certificate establishes the separate Recovery Copy administrative credential/trust domain. It is not by itself counted as a different MFA factor category from TOTP merely because both credentials are present.

The intended authorization path is:

```text
valid RSA-4096 Recovery Copy administrator identity
+
independent administrator authentication/unlock factor
    (for example a PIN/passphrase or equivalent independently classified factor)
+
TOTP step-up factor (30-second v1 step)
+
explicit scoped recovery/export authorization
```

If the RSA private key is hardware-backed and PIN-protected, the implementation records the actual factors it established and does not overstate their classification.

The OTP itself is never stored.

Recovery Copy MFA must remain usable when the normal production Active Directory/Controller environment is unavailable, because loss of the primary environment is a core use case for this system.

## Recovery/export interface

The Recovery Copy administrative interface is separate from replication ingress.

Conceptual allowed actions may include:

```text
ListRecoverableRecoveryPoints
InspectRecoveryCopyVerificationStatus
CreateScopedRecoveryExport
DownloadAuthorizedRecoveryExport
VerifyExportArtifact
```

It does not provide generic operating-system execution or normal direct restore into protected endpoints.

Where practical, replication ingress and recovery administration may use separate network interfaces/VLANs/firewall policy in addition to separate application identities.

## Administrator-controlled export and re-import

The Recovery Copy appliance is not normally mounted as a live Repository by the primary.

The intended disaster-recovery path is:

```text
primary Guidon unavailable/destroyed
    -> administrator authenticates to Recovery Copy under separate trust root + required MFA
    -> select required recovery point(s)
    -> Recovery Copy creates a defined export package
    -> administrator downloads/transfers export
    -> clean/new Guidon appliance receives/imports package
    -> new Guidon independently validates every authoritative artifact
    -> rebuild derived catalog/index
    -> perform supported recovery
```

The receiving Guidon does not trust the export merely because it came from the Recovery Copy appliance.

## Recovery Export package

A future export format should be self-contained for the selected recovery scope and include only the authoritative artifacts required to interpret/verify that recovery data, conceptually:

```text
export descriptor
Repository Format descriptor/reference
selected recovery-point manifest(s)
required immutable objects
recovery-point publication/commit artifact(s)
required authoritative Record v1 history or defined subset
required Journal receipts/checkpoints
required historical Journal public verification material
required trust/configuration provenance
integrity inventory
```

The package format, encryption, segmentation, resumability, and removable-media behavior are frozen during the Recovery Copy implementation phase.

The export mechanism must not silently omit a required artifact and still label the package complete.

## At-rest confidentiality

Recovery Copy storage uses an independent at-rest storage-encryption/key boundary appropriate to the appliance deployment.

Its storage-encryption keys are separate from the primary Repository storage-encryption keys and from Recovery Copy administrator private keys.

Guidon does not assume that copying ciphertext from one ZFS-encrypted dataset automatically provides the intended confidentiality on another host; the Recovery Copy establishes its own storage-at-rest protection.

Durably stored Recovery Copy Jobs/export authorizations follow `../contracts/JOB-STORAGE-AND-MFA-V1.md` and are application-encrypted in addition to any filesystem/storage encryption.

## Root compromise boundary

Root/complete OS compromise of the Recovery Copy appliance is serious. A malicious root may be able to read mounted/decrypted recovery data, inspect process memory, interfere with local services, and access locally available key material.

Guidon does not claim that the Recovery Copy appliance remains confidential or immutable against malicious root merely because its API is push-only or its filesystem is COW.

The security benefit is failure-domain separation:

```text
primary Guidon root compromise
    != automatic Recovery Copy root compromise
    != possession of Recovery Copy administrator RSA-4096 private key
```

Compromise of every relevant host/admin/key authority remains outside Guidon's claim.

## Appliance model

The Recovery Copy system is treated as a Guidon appliance. Normal administrators operate Guidon interfaces, not the underlying OS/storage stack.

Underlying OS/storage access remains an exceptional support/break-glass maintenance path and does not become a normal recovery API.

## Phase placement

Phase 0 freezes the trust/invariant direction in this document so Phase 1 does not create an incompatible Repository format or authority model.

Phase 1 does **not** implement Recovery Copy replication.

Phase 8 is the initial roadmap phase intended to implement and demonstrate the Recovery Copy path.

Phase 9 hardens it for pilot operation, including failure-domain tests, key rotation, capacity behavior, export interruption, and operator procedures.

## Acceptance tests when implemented

At minimum:

- authorized primary can submit an immutable object;
- primary independently cannot read it back using replication credentials;
- primary cannot modify/overwrite an existing conflicting object;
- primary cannot delete Recovery Copy content;
- Recovery Copy independently detects a wrong declared SHA-256;
- Recovery Copy storage loss/partial write does not generate a false verified receipt;
- normal Guidon PKI credential cannot authenticate to recovery-export administration;
- valid separate RSA-4096 recovery administrator certificate plus the required independent authentication factor and TOTP can authorize a scoped export;
- certificate + TOTP alone is not mislabeled MFA if the implementation has not established independent factor categories;
- missing/invalid/replayed MFA fails closed;
- production AD unavailable does not prevent the independent recovery-authentication path;
- exported package imported into a clean Guidon is independently verified before recovery use;
- intentionally corrupted export is rejected;
- original Controller/catalog is not required to interpret a valid export; and
- compromise/loss of primary replication credentials alone does not provide read/delete/export authority.
