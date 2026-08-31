# Guidon

<p align="center">
  <img src="guidon.png" alt="Guidon — Lead the way. Recovery is the mission." width="760">
</p>

<p align="center"><strong>Guidon by Iron Signal Systems</strong><br>
<strong>Lead the way. Recovery is the mission.</strong></p>

> **Status:** Pre-alpha / architecture and engineering foundation.  
> **Production status:** NOT PRODUCTION READY.

Guidon is an Iron Signal Systems backup, recovery, and disaster-recovery project.

Guidon is being engineered around a simple requirement:

> Back up a supported system or workload, recover it at the level required, and be able to trust what was captured, what was recovered, who caused the operation, and how Guidon determined that it succeeded.

Guidon is not intended to compete on the number of integrations, dashboards, or checkboxes it can advertise.

The goal is narrower:

**Support clearly defined workloads and make backup and recovery reliable, secure, inspectable, attributable, and verifiable inside that supported boundary.**

---

## Why Guidon exists

A completed backup job does not by itself prove that a workload can be recovered.

Guidon treats recovery as the mission and backup as one part of that mission.

A protected workload may need to be recovered when the environment around it is partially or completely unavailable. Recovery therefore cannot assume that the original server, operating system, Active Directory trust relationship, local credentials, database instance, controller, or management plane is still healthy.

Guidon is intended to answer questions such as:

- What was backed up?
- When was it captured?
- Is the stored data intact?
- Can it actually be recovered?
- What recovery granularity is available?
- Who requested the operation?
- What identity executed it?
- Where was the data recovered?
- Was the recovered result verified?
- What failed if recovery could not be completed?

---

## Core principles

### Recovery first

Backup creation is not the end goal.

Recovery is.

A completed backup operation does not automatically prove that the protected workload can be restored successfully.

### Trust what Guidon reports

Guidon must not claim more than it knows.

Recovery and verification states are expected to distinguish levels such as:

```text
RECEIVED
STORED
VERIFIED
RECONSTRUCTABLE
RESTORED
VALIDATED
```

These states represent different levels of confidence.

`STORED` must not be reported as `VALIDATED`.

Unknown, unsupported, unavailable, and untested conditions must remain visible.

### Workload-native recovery

Guidon should use supported workload-native consistency and recovery mechanisms where appropriate.

Examples include:

- Windows VSS;
- Microsoft SQL Server native backup and transaction-log mechanisms;
- PostgreSQL base backup, WAL, and logical backup mechanisms.

Guidon should not reimplement complex database internals without a compelling engineering reason.

### Explicit security boundaries

Normal backup execution, human authorization, transport identity, recovery authorization, repository authority, and disaster-recovery access are separate concerns.

Guidon should not collapse them into one highly privileged account or service.

### No arbitrary remote execution

Guidon jobs must describe narrowly defined operations.

Guidon must not become a general-purpose remote shell, PowerShell execution system, or arbitrary command framework.

### Failure must be visible

A failed operation should identify:

- what stage failed;
- what completed successfully;
- what remains usable;
- whether the operation can resume; and
- whether administrative action is required.

Generic success or failure states must not hide meaningful operational information.

---

## Initial workload direction

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
- database recovery;
- alternate-name recovery;
- point-in-time recovery;
- granular database-object recovery where safe and practical.

### PostgreSQL

Planned capabilities include:

- physical base backup;
- WAL protection;
- point-in-time recovery;
- logical backup;
- database recovery;
- schema recovery;
- table recovery.

### Containers

Planned capabilities include:

- workload configuration;
- persistent storage;
- directory recovery;
- individual persistent-file recovery.

---

## Initial repository platform

The initial Guidon repository platform is:

**FreeBSD using Jails**

Guidon may use FreeBSD and ZFS capabilities where they provide useful storage and isolation properties, but the Guidon repository format must remain a Guidon format rather than becoming dependent on one specific underlying filesystem or repository host.

The loss of the original controller or repository host must not automatically make otherwise intact recovery data unusable.

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
Guidon Repository / Control Infrastructure
```

Mutual TLS is a core security requirement.

Each protected endpoint should receive its own Guidon transport identity. Shared endpoint certificates are not the normal deployment model.

The transport identity is separate from the operating-system service identity.

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

## Constrained remote jobs

Remotely initiated Guidon operations must use constrained job objects rather than arbitrary commands.

A job may describe an operation such as:

```text
BACKUP_WINDOWS_VOLUME
BACKUP_MSSQL_DATABASE
RESTORE_FILE
RESTORE_DATABASE
```

A job must not contain a generic executable, shell expression, PowerShell command, script body, or equivalent remote-code-execution field.

Privileged jobs should be:

- typed;
- signed;
- scoped;
- expiring;
- replay resistant;
- attributable.

Endpoints must validate the job before execution.

---

## Human authorization

A valid Guidon job signature does not necessarily prove that the requesting human is authorized to perform the operation.

For Active Directory integrated deployments, Guidon should support independent authorization checks against the requesting user's current AD identity and group membership.

An example administrative group is:

```text
ISS-BK-Admins
```

Internally, authorization should bind to immutable AD identifiers such as SIDs rather than relying only on display names.

A small domain authorization service running under a gMSA may perform read-only Active Directory authorization checks and return a short-lived signed authorization assertion.

A normal manually initiated privileged operation can therefore require both:

```text
valid Guidon job authorization
+
valid current AD authorization
```

Failure of either requirement must cause the normal operation to fail closed and be recorded.

A cryptographically valid job submitted for an unauthorized human is a security-relevant event, not merely a failed backup job.

---

## Windows disaster recovery

Guidon bare-metal recovery must not assume that the environment being recovered is healthy.

For a domain-joined Windows system, recovery must not depend on:

- working Active Directory trust;
- knowledge of the restored local Administrator password;
- availability of the normal gMSA;
- successful domain authentication; or
- the original Windows installation successfully booting before recovery can begin.

Secure-channel repair may be attempted where appropriate, but it is a convenience path rather than a core disaster-recovery dependency.

Guidon is intended to support a controlled first-boot recovery bootstrap for cases where legitimate administrative access to the restored Windows installation is otherwise unavailable.

A temporary recovery identity must be:

- explicitly authorized;
- unique to the recovery;
- randomly credentialed;
- time limited;
- fully audited; and
- removed when recovery is complete.

Guidon must verify cleanup before declaring the recovery fully verified.

---

## Attribution and recovery ledger

Backup and recovery activity must be attributable.

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

Source:
    FS01
    D:\Finance\Budget.xlsx

Recovery point:
    RP-012938

Destination:
    FS01
    D:\Finance\Budget.xlsx

Result:
    VERIFIED
```

The human initiator, machine execution identity, and transport identity are separate facts and must remain separate.

Successful operations and denied privileged operations must be durably recorded.

---

## Recovery data

Guidon recovery points should use versioned, self-describing manifests.

The loss of the normal controller or catalog must not automatically make otherwise intact backup data unrecoverable.

Conceptually:

```text
repository/
    objects/
    manifests/
    recovery-points/
    verification/
    ledger/
```

The catalog may accelerate searching and management, but it must not be the only source capable of interpreting stored recovery data.

---

## Verification

Guidon should independently verify stored backup data.

Repository integrity and workload recoverability are not identical checks.

For example:

```text
object hash valid
```

does not necessarily mean:

```text
SQL database can be restored
```

and:

```text
Windows volume reconstructed
```

does not necessarily mean:

```text
Windows workload is operational
```

Guidon should report these checks separately and never report a stronger verification state than it has actually demonstrated.

---

## Initial engineering target

The first complete vertical slice should remain intentionally narrow:

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

The first implementation should prove that Guidon can:

1. identify a protected Windows source;
2. capture a clearly defined filesystem scope;
3. securely transmit the data;
4. commit a recovery point;
5. independently verify repository objects;
6. enumerate the recovery point;
7. recover an individual file;
8. preserve the supported Windows metadata;
9. verify the recovered file against the stored recovery-point data; and
10. record exactly who requested and executed the recovery.

Only after that path is trustworthy should the supported workload surface expand substantially.

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

Guidon is not initially intended to:

- match incumbent backup vendors feature for feature;
- support every operating system, database, hypervisor, cloud, and application;
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

> Was the recovered result verified?

And when the primary environment itself has failed:

> Can Guidon still lead the workload back to an operational state?

**Lead the way. Recovery is the mission.**
