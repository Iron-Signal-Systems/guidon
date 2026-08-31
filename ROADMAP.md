# Guidon Engineering Roadmap

> **Status:** Planning roadmap for a pre-alpha project.  
> **Purpose:** Sequence engineering work by recoverable capability, not by feature count.

Guidon is developed around one rule:

> **Every major phase must end with a demonstrated recovery path.**

A phase does not need every production optimization before the next phase begins. It does need a small, correct, tested boundary that later work can safely build on.

The planning ranges below are engineering targets, not release promises. Some work will overlap, and security, testing, documentation, and hardening continue throughout the project.

---

## Roadmap summary

| Phase | Engineering target | Primary result |
| --- | ---: | --- |
| 0 — Foundation | 1–2 weeks | Core contracts and trust boundaries frozen enough to implement |
| 1 — Repository core | 3–5 weeks | Durable, verifiable Guidon recovery-point storage |
| 2 — First Windows recovery path | 4–6 weeks | Back up, delete, recover, and verify a Windows file |
| 3 — Windows operational protection and control | 4–8 weeks | Repeatable file/directory protection, signed control, attribution |
| 4 — Windows volume recovery | 6–10 weeks | Reconstruct and verify a protected Windows data volume |
| 5 — Windows bare-metal DR | 10–16 weeks | Rebuild a failed Windows system, including broken-AD cases |
| 6 — Microsoft SQL Server | 10–14 weeks | Full/diff/log protection and verified PITR |
| 7 — PostgreSQL | 10–14 weeks | Base/WAL PITR plus logical granular recovery |
| 8 — Guidon and repository DR | 6–10 weeks | Recover Guidon when controller/catalog/repository components fail |
| 9 — Pilot hardening | 4–8 months | Installation, upgrades, scale, abuse testing, runbooks, pilot readiness |

A reasonable planning target is:

```text
First verified Windows file recovery:     ~2–3 months
Windows bare-metal recovery:              ~7–11 months
Windows + MSSQL + PostgreSQL capability:  ~12–17 months
Controlled pilot candidate:               ~18–24 months
Broader external pilot readiness:         ~24–30 months
```

These are cumulative engineering ranges, not contractual dates.

---

# Phase 0 — Foundation

**Target:** 1–2 weeks

The goal is not to design the entire future product. The goal is to freeze only the contracts that would be expensive or dangerous to discover accidentally during implementation.

## Define

- recovery-point identity;
- repository object identity;
- versioned manifest envelope;
- hash/integrity rules;
- object commit semantics;
- recovery-point commit semantics;
- minimum verification-state meanings;
- endpoint identity model;
- mTLS trust and revocation model;
- constrained job envelope;
- operation/recovery ledger envelope;
- human, execution, and transport identity separation;
- FreeBSD Jail responsibility boundaries; and
- initial threat model.

## Explicitly defer

- global deduplication optimization;
- repository replication;
- high availability;
- polished UI;
- advanced retention policy;
- mass deployment tooling;
- broad workload abstractions; and
- speculative plugin/framework architecture.

## Exit gate

Guidon can clearly answer:

1. What is a repository object?
2. When is an object durable?
3. What is a recovery point?
4. When is a recovery point committed?
5. How is integrity checked?
6. How is a protected endpoint identified?
7. How could a future Guidon installation interpret an intact recovery point without the original catalog?

---

# Phase 1 — Repository core

**Target:** 3–5 weeks

Build the smallest correct FreeBSD repository service.

## Implement

- FreeBSD host baseline;
- initial Jail boundary;
- HTTPS listener on TCP 443;
- mutual TLS;
- unique endpoint identity validation;
- streamed object upload;
- content hashing while receiving;
- temporary-object write;
- durable commit boundary;
- duplicate-object handling;
- versioned manifest submission;
- atomic recovery-point commit;
- object existence/integrity verification;
- recovery-point enumeration; and
- minimal append-oriented operation records.

The initial data path should stay conceptually simple:

```text
receive
    -> hash
    -> write temporary object
    -> durable storage boundary
    -> commit object
    -> reference from manifest
    -> commit recovery point
```

## Failure tests

- disconnect halfway through an object;
- kill the repository process during receive;
- resend the same object;
- submit a manifest referencing a missing object;
- corrupt a committed object;
- exhaust repository storage; and
- restart the Jail and re-enumerate committed recovery points.

## Exit gate

Guidon can accept arbitrary test objects over mTLS, durably commit them, create a recovery point, independently detect corruption, and refuse to create a false-success recovery point when required data is incomplete.

---

# Phase 2 — First Windows recovery path

**Target:** 4–6 weeks

This phase proves the product idea end to end.

## Implement

- Windows service skeleton;
- domain-joined gMSA execution;
- unique endpoint mTLS identity;
- protected-source identity;
- explicit filesystem scope;
- file enumeration;
- content capture;
- minimum supported NTFS metadata;
- repository transfer;
- manifest construction;
- recovery-point commit;
- recovery-point browsing;
- alternate-location file restore;
- original-location file restore; and
- restored-content verification.

VSS should be introduced where consistency requires it, but this phase should not be expanded into full-system backup work.

## Canonical acceptance test

```text
create file
    -> back up
    -> verify repository
    -> delete file
    -> recover file
    -> verify content
    -> verify supported metadata
    -> inspect operation record
```

Repeat with interrupted transfer and repository restart.

## Exit gate

Guidon can truthfully demonstrate:

> A Windows file was captured, stored, verified, destroyed, recovered, and verified against its recovery point.

Expected cumulative target: **approximately 2–3 months from project implementation start.**

---

# Phase 3 — Windows operational protection and control

**Target:** 4–8 weeks

Turn the first vertical slice into a repeatable operational path without jumping to bare-metal recovery yet.

## Implement

- directory protection;
- larger file sets;
- checkpoint/resume behavior;
- VSS-backed filesystem consistency where required;
- richer supported NTFS metadata;
- recovery browsing;
- scheduled backup jobs;
- manual backup jobs;
- constrained signed job objects;
- expiration and replay protection;
- endpoint/job binding;
- human identity attribution;
- AD authorization broker under gMSA;
- SID-based authorization;
- fail-closed authorization behavior;
- denied-operation records; and
- minimal operational UI or CLI needed to exercise the system safely.

## Exit gate

An authorized administrator can start a constrained backup or restore operation without granting Guidon arbitrary remote-command capability, and Guidon can show who requested the operation, what identity executed it, and what actually occurred.

---

# Phase 4 — Windows volume recovery

**Target:** 6–10 weeks

Expand from files/directories to data-volume reconstruction.

## Implement and validate

- volume capture model;
- VSS coordination;
- directory tree reconstruction;
- ACL and ownership preservation;
- alternate data streams;
- sparse-file behavior;
- reparse-point behavior;
- hard-link behavior where supported;
- large file sets;
- millions-of-file characterization;
- restore to alternate disk/volume;
- restore to replacement volume; and
- post-recovery verification.

## Exit gate

Destroy or replace a protected Windows data volume and reconstruct it from Guidon with supported content and metadata verified.

---

# Phase 5 — Windows bare-metal disaster recovery

**Target:** 10–16 weeks

This is the largest Windows engineering phase and a major product milestone.

## Implement

- Guidon recovery environment;
- repository authentication from recovery environment;
- disk and partition reconstruction;
- EFI/system/boot volume recovery;
- Windows system-volume recovery;
- BCD/boot reconstruction as required;
- data-volume integration;
- first-boot validation;
- workload/service validation; and
- recovery ledger integration.

## Broken-domain recovery requirement

Bare-metal recovery must not depend on:

- working AD trust;
- knowing the restored local Administrator password; or
- the normal gMSA being immediately usable.

Secure-channel repair may be attempted but is not the required recovery mechanism.

Implement the controlled first-boot recovery bootstrap:

- signed recovery authorization;
- target-machine binding;
- expiration;
- nonce/replay protection;
- one-time temporary local recovery administrator;
- unique random credential;
- tightly controlled credential release;
- TTL/watchdog cleanup;
- account removal; and
- verification that the bootstrap is disarmed before final validation.

## Signature acceptance test

Create a recovery scenario where:

```text
machine account password in AD != restored machine state
local Administrator password is unavailable
normal gMSA cannot initially authenticate
```

Destroy the protected machine and recover it to a verified operational state.

## Exit gate

Guidon can rebuild a failed Windows system and regain controlled administrative operation without requiring healthy pre-existing AD trust or a known restored local Administrator credential.

Expected cumulative target: **approximately 7–11 months.**

---

# Phase 6 — Microsoft SQL Server

**Target:** 10–14 weeks

Guidon should orchestrate Microsoft SQL Server's supported native mechanisms rather than reimplementing database internals.

## Implement

- instance/database discovery;
- least-privilege execution model;
- full backup;
- differential backup;
- transaction-log backup;
- backup-chain metadata;
- recovery-model awareness;
- LSN relationships;
- chain completeness checks;
- complete database restore;
- alternate-name/location restore;
- point-in-time restore;
- restore validation;
- isolated restore testing; and
- operation attribution.

## Failure tests

- missing differential;
- missing log backup;
- broken log chain;
- interrupted transfer;
- repository object corruption;
- unavailable target instance;
- insufficient SQL permission;
- alternate recovery destination; and
- recovery to a point between log backups.

## Later within the phase, if practical

Granular object/table recovery should use an isolated temporary database restore and supported extraction path rather than direct MDF/page manipulation.

## Exit gate

Destroy a test database and recover it to a selected point in time. Open it, validate expected data, and record exactly how the recovery was constructed and executed.

---

# Phase 7 — PostgreSQL

**Target:** 10–14 weeks

PostgreSQL requires both physical and logical protection paths because they solve different recovery problems.

## Physical protection

- cluster discovery;
- physical base backup;
- WAL archive/stream handling;
- WAL continuity tracking;
- timeline handling;
- point-in-time recovery;
- alternate recovery location; and
- recovered-cluster validation.

## Logical protection

- logical database backup;
- schema backup/recovery;
- table backup/recovery; and
- granular recovery validation.

## Failure tests

- missing WAL segment;
- interrupted base backup;
- corrupt base-backup object;
- timeline changes;
- insufficient PostgreSQL permission;
- unavailable recovery target; and
- incompatible recovery request.

## Exit gate

Recover an entire PostgreSQL cluster to a selected point in time and independently recover a selected logical object such as a table.

Expected cumulative target for Windows + MSSQL + PostgreSQL capability: **approximately 12–17 months.**

---

# Phase 8 — Guidon and repository disaster recovery

**Target:** 6–10 weeks

The backup system must itself be recoverable.

## Test and implement recovery from

- controller database loss;
- controller host loss;
- repository-service loss;
- repository-host replacement;
- destroyed indexes/catalogs; and
- software version changes.

## Required capabilities

- import/mount intact repository data;
- enumerate self-describing recovery points;
- rebuild catalog/index data;
- verify manifests and referenced objects;
- distinguish unavailable data from corrupt data; and
- recover protected workloads without requiring the original controller database.

Repository replication may be introduced here if earlier scale or resilience testing justifies it.

## Exit gate

Destroy the Guidon controller/catalog, rebuild Guidon around intact repository data, rediscover recovery points, verify them, and perform a real workload recovery.

---

# Phase 9 — Pilot hardening

**Target:** 4–8 months

This phase turns demonstrated recovery capability into something appropriate for another organization to operate in a controlled pilot.

Hardening work begins before Phase 9; this phase closes the remaining gaps.

## Areas

### Deployment and lifecycle

- installers;
- configuration validation;
- service lifecycle;
- certificate enrollment/rotation/revocation;
- upgrades;
- rollback;
- backward recovery-point compatibility; and
- uninstallation/cleanup behavior.

### Repository operations

- capacity monitoring;
- retention;
- garbage collection;
- scrubbing;
- storage-full behavior;
- compression/deduplication where justified;
- repository migration; and
- replication/immutability options where justified.

### Scale

- concurrent backup jobs;
- concurrent restore jobs;
- large files;
- millions of files;
- long-running backups;
- slow links;
- interrupted links;
- hundreds of protected systems; and
- realistic repository growth.

### Security

- threat-model reassessment;
- privilege review;
- mTLS/key lifecycle review;
- authorization abuse tests;
- replay/tamper tests;
- input fuzzing where applicable;
- recovery-environment review;
- bootstrap cleanup abuse cases; and
- independent code/security review.

### Operations

- useful UI/CLI workflows;
- alerting;
- metrics;
- logs;
- SIEM/syslog integration where useful;
- operator runbooks;
- recovery runbooks;
- failure diagnostics; and
- documented support boundaries.

### Recovery drills

Repeatedly destroy and recover:

- files;
- directories;
- Windows volumes;
- Windows systems;
- Windows systems with broken AD trust;
- SQL Server databases;
- PostgreSQL clusters;
- Guidon controller/catalog state; and
- repository service infrastructure.

## Exit gate

A pilot organization can install Guidon, operate supported protection jobs, understand failures, perform documented recoveries, and independently confirm what Guidon claims about those recoveries.

A reasonable target is **approximately 18–24 months for a tightly controlled pilot candidate** and **24–30 months for broader external pilot readiness**, assuming the scope remains disciplined and no major platform blocker forces redesign.

---

# Scope discipline

The current active workload boundary is:

```text
Windows Server / Workstation
Microsoft SQL Server
PostgreSQL
```

The roadmap does **not** currently include:

- containers;
- Kubernetes;
- VMware;
- Hyper-V;
- Microsoft 365;
- Exchange;
- Oracle;
- cloud-native workload families; or
- broad plugin/framework support.

New workload families should be added only after the existing recovery paths justify expanding the product boundary.

---

# Roadmap principle

The roadmap is deliberately ordered so that Guidon earns each stronger claim.

```text
store bytes
    -> prove integrity
    -> recover a file
    -> recover a volume
    -> recover a machine
    -> recover the machine when its surrounding identity infrastructure is damaged
    -> recover databases to a point in time
    -> recover Guidon itself
    -> prove the whole system under sustained failure and scale
```

Guidon should never advance a capability because the backup side merely appears to work.

**Recovery remains the mission.**
