# Guidon

<p align="center">
  <img src="guidon.png" alt="Guidon — Lead the way. Recovery is the mission." width="760">
</p>

<p align="center"><strong>Guidon by Iron Signal Systems</strong><br>
<strong>Lead the way. Recovery is the mission.</strong></p>

> **Status:** Pre-alpha / engineering foundation.  
> **Production status:** NOT PRODUCTION READY.

Guidon is an Iron Signal Systems backup, recovery, and disaster-recovery project.

Guidon is built around a simple requirement:

> **Back up a supported system or workload, recover it at the level required, and be able to trust what was captured, what was recovered, who caused the operation, and how Guidon determined that it succeeded.**

Backup creation is not the mission.

**Recovery is.**

---

## Product scope

The current Guidon scope is intentionally narrow.

### Windows Server and Workstation

Planned recovery levels include:

- full system;
- volume;
- directory;
- individual file.

### Microsoft SQL Server

Planned capabilities include:

- full database backup;
- differential backup;
- transaction-log backup;
- full database recovery;
- alternate-name/location recovery;
- point-in-time recovery; and
- granular database-object recovery where safe and practical.

### PostgreSQL

Planned capabilities include:

- physical base backup;
- WAL protection;
- point-in-time recovery;
- logical backup;
- database recovery;
- schema recovery; and
- table recovery.

Containers, Kubernetes, hypervisors, cloud workloads, Microsoft 365, Exchange, Oracle, and other workload families are outside the current project scope.

---

## Core principles

### Recovery first

A completed backup job does not prove that a workload can be recovered.

Each major Guidon development phase must end in a recovery demonstration, not merely a backup demonstration.

### Trust what Guidon reports

Guidon must not claim more than it knows.

Recovery and verification states may distinguish levels such as:

```text
RECEIVED
STORED
VERIFIED
RECONSTRUCTABLE
RESTORED
VALIDATED
```

These states are not interchangeable.

For example, `STORED` must not be reported as `VALIDATED`.

Unknown, unsupported, unavailable, incomplete, and untested conditions must remain visible.

### Workload-native recovery

Guidon should use supported workload-native consistency and recovery mechanisms where appropriate.

Examples include:

- Windows VSS;
- Microsoft SQL Server native backup and transaction-log mechanisms; and
- PostgreSQL base backup, WAL, and logical backup mechanisms.

Guidon should orchestrate, protect, verify, recover, and record these operations rather than unnecessarily reimplementing complex workload internals.

### Simple components, explicit boundaries

Guidon should prefer small, understandable components over generalized frameworks.

Complexity must solve a demonstrated problem.

Normal backup execution, human authorization, transport identity, repository authority, recovery authorization, and disaster-recovery access are separate concerns and must not silently collapse into one highly privileged identity.

### No arbitrary remote execution

Remotely initiated Guidon work must use constrained, typed operations.

Guidon must not become a general-purpose remote shell, PowerShell execution system, arbitrary script runner, or generic remote-code-execution framework.

### Failure is part of the product

A failed operation should identify:

- what stage failed;
- what completed successfully;
- what remains usable;
- whether the operation can resume; and
- whether administrative action is required.

Generic success or failure states must not hide meaningful operational information.

---

## Initial architecture direction

The first complete Guidon path is intentionally small:

```text
Domain-joined Windows Server
        |
        | Guidon service using gMSA
        |
        | outbound mTLS / TCP 443
        v
FreeBSD / Jail
        |
        v
Guidon Repository
```

The initial repository platform is **FreeBSD using Jails**.

Guidon may use FreeBSD and ZFS capabilities where useful, but the Guidon recovery format must remain a Guidon format rather than becoming dependent on a specific underlying filesystem or original controller.

The first vertical slice must prove that Guidon can:

1. identify a protected Windows source;
2. capture a clearly defined filesystem scope;
3. securely transmit the data;
4. durably commit repository objects;
5. commit a versioned recovery point;
6. independently verify the stored objects;
7. enumerate the recovery point;
8. recover an individual file;
9. verify the recovered file against the stored recovery point; and
10. record what happened and which identities were involved.

The target for this first complete recovery path is measured in **weeks**, not months of pre-implementation architecture work.

---

## Transport and endpoint identity

Protected systems should normally initiate connections to Guidon infrastructure.

The initial transport direction is:

```text
Protected System
        |
        | outbound
        | mTLS
        | TCP 443
        v
Guidon Infrastructure
```

Mutual TLS is a core security requirement.

Each protected endpoint should have its own Guidon transport identity. Shared endpoint certificates are not the normal deployment model.

Transport identity is separate from the operating-system service identity.

---

## Windows service identity

For domain-joined Windows systems:

> **Normal Guidon backup services must use a Group Managed Service Account (gMSA).**

Stored domain service-account passwords are not an acceptable normal operating model.

Example:

```text
Windows execution identity:
    DOMAIN\gGuidon-BK-FS01$

Guidon transport identity:
    unique mTLS client identity for FS01
```

Guidon should use the least Windows privileges required for each service boundary and must not require Domain Admin membership merely to perform normal backup operations.

---

## Constrained jobs and human authorization

A remotely initiated operation must be represented as a constrained job object, not an arbitrary command.

Examples include:

```text
BACKUP_WINDOWS_VOLUME
BACKUP_MSSQL_DATABASE
RESTORE_FILE
RESTORE_DATABASE
```

Privileged jobs should be:

- typed;
- signed;
- scoped;
- expiring;
- replay resistant; and
- attributable.

A valid Guidon job signature does not by itself prove that the requesting human is authorized to perform the operation.

For Active Directory integrated deployments, Guidon should support an independent authorization decision based on the requesting user's current AD identity and authorization membership.

Internally, authorization should bind to immutable identifiers such as SIDs rather than relying only on display names.

A normal manually initiated privileged operation may therefore require both:

```text
valid Guidon job authorization
+
valid current AD authorization
```

Failure of either requirement must cause the normal operation to fail closed and be recorded.

---

## Windows disaster recovery

Guidon bare-metal recovery must not assume that the environment being recovered is healthy.

For a domain-joined Windows system, recovery must not depend on:

- working Active Directory trust;
- knowledge of the restored local Administrator password;
- availability of the normal gMSA;
- successful domain authentication; or
- the original Windows installation successfully booting before recovery can begin.

Secure-channel repair may be attempted where useful, but it is a convenience path rather than a core disaster-recovery dependency.

Guidon is intended to support a tightly controlled first-boot recovery bootstrap for cases where legitimate administrative access to the restored Windows installation is otherwise unavailable.

Any temporary recovery identity must be:

- explicitly authorized;
- unique to the recovery;
- randomly credentialed;
- time limited;
- fully audited; and
- removed when recovery is complete.

Guidon must verify cleanup before declaring that recovery fully validated.

---

## Attribution and recovery ledger

Guidon must distinguish the person who requested an operation from the service or machine identity that executed it.

Conceptually:

```text
Requested by:
    DOMAIN\jwood

Executed by:
    DOMAIN\gGuidon-BK-FS01$

Transport identity:
    FS01 Guidon mTLS client

Operation:
    FILE_RESTORE

Recovery point:
    RP-012938

Result:
    VERIFIED
```

The human initiator, execution identity, and transport identity are separate facts and must remain separate.

Successful operations and denied privileged operations must be durably recorded.

---

## Recovery data

Guidon recovery points should use versioned, self-describing manifests.

The loss of the normal controller or catalog must not automatically make otherwise intact recovery data unusable.

Conceptually:

```text
repository/
    objects/
    manifests/
    recovery-points/
    verification/
    ledger/
```

The catalog may accelerate searching and management, but it must not be the only component capable of interpreting stored recovery data.

---

## Verification

Repository integrity and workload recoverability are different checks.

For example:

```text
object hash valid
```

is not the same claim as:

```text
SQL database can be restored
```

Likewise:

```text
Windows volume reconstructed
```

is not automatically the same as:

```text
Windows workload validated as operational
```

Guidon must report the level it has actually demonstrated.

---

## Engineering roadmap

Guidon is developed by completing progressively larger recovery paths.

The current sequence is:

```text
FOUNDATION
    |
    v
REPOSITORY CORE
    |
    v
WINDOWS FILE RECOVERY
    |
    v
WINDOWS OPERATIONAL PROTECTION / CONTROL
    |
    v
WINDOWS VOLUME RECOVERY
    |
    v
WINDOWS BARE-METAL DR
    |
    v
MICROSOFT SQL SERVER
    |
    v
POSTGRESQL
    |
    v
GUIDON / REPOSITORY DR
    |
    v
PILOT HARDENING
```

The detailed engineering phases, exit criteria, and planning ranges are maintained in [ROADMAP.md](ROADMAP.md).

---

## Failure-driven engineering

Guidon should be developed against failure scenarios rather than only successful demonstrations.

Important scenarios include:

- network connection loss during backup;
- repository process crash;
- protected-agent crash;
- repository storage exhaustion;
- duplicate object upload;
- corrupt stored object;
- missing repository object;
- controller database loss;
- controller host loss;
- older recovery point restored by newer Guidon software;
- Windows server restored with broken AD trust;
- unavailable restored local Administrator credential;
- unavailable gMSA during early disaster recovery;
- interrupted recovery;
- interrupted temporary-recovery-account cleanup;
- incomplete Microsoft SQL Server recovery chain; and
- incomplete PostgreSQL WAL chain.

Behavior under these conditions is part of the product.

---

## What Guidon is not

Guidon is not currently intended to:

- match incumbent backup vendors feature for feature;
- support every operating system, database, hypervisor, cloud, and application;
- support containers or Kubernetes;
- provide arbitrary remote administration;
- hide unsupported or unverified conditions behind a green status;
- require broadly privileged domain identities everywhere;
- make recovery dependent on an opaque controller database; or
- add complexity merely to make the product appear sophisticated.

A smaller supported surface that can be trusted is preferable to broad nominal support that cannot be confidently recovered.

---

## Definition of success

Guidon succeeds when an administrator can confidently answer:

> What do I have?

> Is it intact?

> Can I recover it?

> Can I recover only the thing I need?

> Who initiated the operation?

> What identity actually performed it?

> What happened during recovery?

> Was the recovered result actually verified?

And when the primary environment itself has failed:

> Can Guidon still lead the workload back to an operational state?

**Lead the way. Recovery is the mission.**
