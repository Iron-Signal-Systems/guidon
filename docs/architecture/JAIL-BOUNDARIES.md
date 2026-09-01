# FreeBSD Jail Boundaries

## Principle

A Guidon component receives only the filesystem, network, key material, and operating-system privilege required for its specific responsibility.

Sharing a FreeBSD host does not justify sharing authority.

## Initial layout

The minimum application layout is:

```text
FreeBSD Host
│
├── guidon-repository Jail
│
└── guidon-journal Jail
```

Guidon should not multiply Jails/services merely to create a microservice architecture. New boundaries must solve a demonstrated security, privilege, failure-isolation, or operational problem.

## Repository Jail

The Repository Jail owns recovery semantics and may access:

```text
Repository objects
recovery-point manifests
recovery-point commit markers
Repository authoritative Record v1 files
Journal receipts returned to Repository
temporary incoming objects
Repository configuration required for operation
```

It may perform defined operations such as:

- receive/hash/verify/publish objects;
- build/accept/verify manifests;
- prepare/publish recovery points;
- enumerate recovery data;
- serve authorized recovery content;
- create authoritative Guidon records;
- submit exact records to Journal; and
- perform authorized retention/GC actions.

It must not possess:

- Journal signing private key;
- Controller Job signing private key;
- AD Authorization Broker signing private key;
- Recovery Authority private key; or
- protected-endpoint private keys.

It must not directly write Journal datasets.

## Journal Jail

The Journal Jail owns:

```text
Journal streams
Journal entries
Journal segments
Journal checkpoints
Journal receipts
Journal public verification-key history
Journal attestation signing private key
```

It may:

- authenticate authorized Guidon producers;
- receive exact Record v1 bytes;
- calculate SHA-256 independently;
- assign stream/sequence/entry identities;
- durably append Journal entries;
- create signed receipts;
- seal segments;
- create/sign checkpoints; and
- perform Journal integrity/reconciliation work.

It must not:

- browse customer backup objects;
- restore workload data;
- publish recovery points;
- delete recovery points;
- issue Controller jobs;
- authorize AD users;
- execute endpoint operations;
- modify Repository authoritative record bytes; or
- access Recovery Authority private keys.

The Journal understands only the common Record v1 envelope needed to validate/attest submissions. It does not become a backup-policy engine.

## Filesystem visibility

Filesystem mounts enforce separation; application promises alone are insufficient.

Conceptually:

```text
Repository Jail sees:
    /guidon/repository/objects
    /guidon/repository/recovery-points
    /guidon/repository/records
    /guidon/repository/journal-receipts

Journal Jail sees:
    /guidon/journal/streams
    /guidon/journal/checkpoints
    /guidon/journal/keys
```

The Repository Jail must not receive even read-only access to the Journal private-key location.

The Journal Jail does not mount the Repository merely to inspect manifests or make recovery decisions.

## Service identities

Guidon application services inside Jails should normally run under dedicated least-privilege service users rather than UID 0.

Conceptually:

```text
guidonrepo
guidonjournal
```

If an operation later proves to require elevated OS privilege, prefer a narrow purpose-specific helper over running the entire service as root.

For Journal signing, the preferred direction is a constrained signer boundary so that a network/writer process can request only defined Journal signature operations rather than reading raw private-key bytes or calling a generic signing oracle.

## Network exposure

The Repository Jail exposes only the Guidon listeners required for protected endpoint and authorized infrastructure operation.

Protected endpoints do not receive generic SSH, SMB, NFS, or arbitrary management access merely for Guidon backup.

The Journal listener is restricted to explicitly authorized Guidon producers. A certificate that is valid under the CA but not authorized as a Journal producer is rejected.

Repository-to-Journal traffic remains mTLS protected even on the same host and is subject to the same PCAP/Wireshark acceptance testing as other Guidon-controlled connections.

## Separate transport/signing credentials

Repository and Journal have separate stable component identities and separate transport certificates.

The Journal transport certificate/private key is separate from the Journal Ed25519 attestation signing key.

## Host responsibilities

The FreeBSD host should remain infrastructure-focused:

```text
ZFS pool/dataset management
Jail lifecycle
network interfaces/bridges
host time synchronization
hardware/storage monitoring
base OS patching
```

Guidon backup semantics, Journal semantics, workload parsing, recovery engines, and application databases belong inside defined Guidon component boundaries rather than becoming host scripts/processes.

## Dataset ownership and capacity separation

Initial direction:

```text
guidon/repository/objects             -> Repository authority
guidon/repository/recovery-points     -> Repository authority
guidon/repository/records             -> Repository authority

guidon/journal/streams                -> Journal authority
guidon/journal/checkpoints            -> Journal authority
guidon/journal/keys                   -> Journal signing boundary
```

Separate dataset quotas/reservations should prevent ordinary Repository growth from immediately consuming all capacity needed for Journal history. Exact capacity values are deployment/implementation decisions.

If Journal durable storage is exhausted, gated operations stop rather than deleting old Journal history to keep operating.

## Failure independence

A Repository process/Jail failure does not require the Journal to fail. Previously attested Journal history remains available.

A Journal failure does not require the Repository to discard valid incoming backup data. Durable-asynchronous operations may continue while defined Journal-gated transitions remain blocked.

## Host-root limitation

Jails are a meaningful component and privilege-isolation boundary, but they are not a defense against complete FreeBSD host-root compromise.

The accurate claim is:

```text
Repository Jail compromise != Journal Jail compromise
Journal Jail compromise != Repository Jail compromise
FreeBSD host-root compromise can potentially cross both boundaries
```

Future HSM-backed signing, external checkpoint anchoring, or a physically separate Journal may strengthen the trust boundary if later justified. Guidon must not claim those protections before they exist.
