# FreeBSD Jail Boundaries

## Principle

A Guidon component receives only the filesystem, network, key material, and operating-system privilege required for its specific responsibility.

Sharing a FreeBSD host does not justify sharing authority.

The customer operates a **Guidon appliance**. Guidon owns the normal lifecycle of the FreeBSD/ZFS/Jail platform beneath it.

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

## Appliance operational model

Guidon is not intended to require normal administrators to become FreeBSD/ZFS/Jail administrators.

Normal operations are exposed through Guidon-managed interfaces for tasks such as:

```text
initial appliance configuration
network configuration
storage enrollment/status
protected-system enrollment
backup/recovery policy
Journal/recovery status
Guidon updates
health/alerting
support bundle generation
recovery operations
```

Routine product operation must not require administrators to manually:

```text
pkg update
edit pf.conf
manage Jail configuration
manage Guidon ZFS dataset layout
repair Guidon application permissions by hand
manually rotate Guidon service certificates/keys
SSH into the host to determine ordinary product health
```

The underlying FreeBSD/ZFS/Jail implementation remains visible in technical architecture/support documentation because its actual security/failure properties matter. It is not hidden by marketing language, but it is managed as part of the appliance rather than delegated to the customer as a second product to operate.

## Support and break-glass OS access

Guidon must not make the appliance abstraction an obstacle to emergency diagnosis or recovery.

A controlled underlying OS/support path may exist for:

```text
console/SSH diagnostics
zpool/ZFS inspection
network diagnostics
packet capture
hardware/storage inspection
low-level repair under explicit support procedure
```

This is an exceptional maintenance boundary, not the normal administrator API.

Where Guidon can observe it, entry into an elevated support/break-glass maintenance mode should be explicit and auditable. The existence of such a path does not change the host-root limitation below.

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
- Recovery Authority private key;
- Recovery Copy administrator private key; or
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
- modify Repository authoritative record bytes;
- access Recovery Copy recovery data; or
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

Future separate Journal/Recovery Copy/External Witness appliances retain the same no-plaintext-fallback principle but use their own defined identities and authorities rather than inheriting System 1 credentials automatically.

## Separate transport/signing credentials

Repository and Journal have separate stable component identities and separate transport certificates.

The Journal transport certificate/private key is separate from the Journal Ed25519 attestation signing key.

## Host responsibilities

The FreeBSD host remains infrastructure-focused:

```text
ZFS pool/dataset management
Jail lifecycle
network interfaces/bridges
host time synchronization
hardware/storage monitoring
base OS patching
```

Guidon backup semantics, Journal semantics, workload parsing, recovery engines, and application databases belong inside defined Guidon component boundaries rather than becoming host scripts/processes.

Under the appliance model, **Guidon owns the normal management of these host responsibilities**. The fact that FreeBSD performs the function does not make the customer responsible for manually operating it.

## Appliance update responsibility

Guidon appliance updates must eventually own the complete supported update path, including as applicable:

```text
signed update artifact
preflight/compatibility checks
configuration/state protection
controlled service/Jail lifecycle
OS + Guidon component update
schema/config migration
restart
health/integrity validation
rollback/failure handling
```

An appliance update must not silently make previously valid Repository Format generations unreadable.

Recovery compatibility across supported versions is a product requirement, not something delegated to manual FreeBSD package management.

## Hardware/storage health surfaces

Guidon should surface relevant underlying health facts through the appliance rather than requiring administrators to discover them manually.

Examples include:

```text
ZFS pool state
checksum/storage errors
scrub results
capacity pressure
failed/degraded devices
I/O failures
SMART/device observations where available
required post-failure verification epoch
```

Guidon must distinguish an observed storage condition from an unsupported conclusion. A configured mirror is not automatically labeled healthy without the defined observation.

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

Future independent Journal B, External Witness, and Recovery Copy systems are separate appliance/failure domains as defined in their architecture documents. Merely placing two Jails on one physical host is never described as equivalent to separate-host failure independence.

## Host-root limitation

Jails are a meaningful component and privilege-isolation boundary, but they are not a defense against complete FreeBSD host-root compromise.

The accurate claim is:

```text
Repository Jail compromise != Journal Jail compromise
Journal Jail compromise != Repository Jail compromise
FreeBSD host-root compromise can potentially cross both boundaries on that host
```

The appliance abstraction does not change this fact. An attacker/support operator who obtains true host root has authority below the Jail/application boundary and may potentially manipulate software, storage, process memory, mounted data, and locally available keys.

Separately hosted Journal witnesses, an External Witness, and a Recovery Copy appliance may strengthen the overall failure/trust architecture because compromise of System 1 root does not automatically grant root/keys on those independent systems.

Guidon must not claim those protections unless the independent systems are actually deployed and their boundaries tested.
