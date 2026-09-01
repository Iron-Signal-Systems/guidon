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
| 1 — Repository core | 3–5 weeks | Durable, journal-attested, verifiable Guidon recovery-point storage |
| 2 — First Windows recovery path | 4–6 weeks | Back up, delete, recover, and verify a Windows file |
| 3 — Windows operational protection and control | 4–8 weeks | Repeatable file/directory protection, signed control, attribution |
| 4 — Windows volume recovery | 6–10 weeks | Reconstruct and verify a protected Windows data volume |
| 5 — Windows bare-metal DR | 10–16 weeks | Rebuild a failed Windows system, including broken-AD cases |
| 6 — PostgreSQL | 10–14 weeks | Base/WAL PITR plus logical granular recovery |
| 7 — Microsoft SQL Server | 10–14 weeks | Full/diff/log protection and verified PITR |
| 8 — Guidon and repository DR | 6–10 weeks | Recover Guidon when controller/catalog/repository components fail |
| 9 — Pilot hardening | 4–8 months | Installation, upgrades, scale, abuse testing, runbooks, pilot readiness |

A reasonable planning target is:

```text
First verified Windows file recovery:          ~2–3 months
Windows bare-metal recovery:                   ~7–11 months
Windows + PostgreSQL + MSSQL capability:       ~12–17 months
Controlled pilot candidate:                    ~18–24 months
Broader external pilot readiness:              ~24–30 months
```

These are cumulative engineering ranges, not contractual dates.

---

# Phase 0 — Foundation

**Target:** 1–2 weeks

The goal is not to design the entire future product. The goal is to freeze only the contracts that would be expensive or dangerous to discover accidentally during implementation.

## Define

- repository object identity and immutable SHA-256 naming;
- versioned manifest envelope;
- UUIDv7 recovery-point identity;
- object and recovery-point durability/commit semantics;
- ZFS durability requirements and prohibition on `sync=disabled` for Guidon durable datasets;
- restart/crash reconciliation behavior;
- mandatory full post-failure reverification after unclean shutdown, uncertain shutdown, storage fault, or power loss;
- verification-state meanings;
- Record v1 factual-history contract;
- Journal stream, receipt, checkpoint, signing, durability, and idempotency contracts;
- synchronous Journal-gated versus durable-asynchronous records;
- endpoint, user, gMSA, service, transport, and producer identity separation;
- endpoint PKI lifecycle and certificate binding;
- constrained Job v1 envelope;
- AD Authorization Assertion v1;
- break-glass Recovery Authority boundary;
- observed-only network attribution model;
- mTLS trust and revocation model;
- PCAP/Wireshark transport acceptance requirements;
- retention and deletion authority contract;
- configuration/policy provenance contract;
- time/clock provenance including ISS-system clock observation;
- FreeBSD Repository/Journal Jail responsibility boundaries; and
- initial threat model and explicit unsupported claims.

The detailed contracts are maintained under [`docs/`](docs/README.md).

## Explicitly defer

- chunk-size/chunking optimization beyond the required object contract;
- repository replication;
- high availability;
- polished UI;
- advanced retention-policy features beyond the deletion-safety contract;
- mass deployment tooling;
- broad workload abstractions;
- speculative plugin/framework architecture; and
- hardware-backed/external Journal anchoring unless implementation evidence justifies it early.

## Exit gate

Guidon can clearly answer:

1. What is a repository object?
2. When is an object durable?
3. What is a recovery point?
4. When is a recovery point committed?
5. What does each verification/recovery state actually mean?
6. How is a protected endpoint identified when certificates rotate?
7. Which authority may request, authorize, attest, recover, or destroy data?
8. What must happen after a crash or power loss before old backups are presented as currently verified?
9. How is authoritative operational history independently journal-attested?
10. How can a future Guidon installation interpret intact recovery data without the original catalog?
11. What security claims does Guidon make, and where do those claims explicitly stop?

---

# Phase 1 — Repository core

**Target:** 3–5 weeks

Build the smallest correct FreeBSD Repository and Journal path.

## Implement

- FreeBSD host baseline;
- separate Repository and Journal Jails;
- dedicated least-privilege service identities;
- separate ZFS datasets and durability boundaries;
- Repository HTTPS listener on TCP 443;
- Repository-to-Journal mTLS;
- mutual TLS identity binding;
- unique endpoint identity validation;
- streamed object upload;
- SHA-256 while receiving;
- temporary-object write;
- explicit synchronous durability boundary;
- atomic immutable object publication;
- duplicate-object independent verification;
- versioned manifest submission;
- recovery-point preparation and publication contract;
- authoritative Record v1 storage;
- Journal exact-byte record receipt, independent SHA-256, stream sequence, and signed receipt;
- bounded Journal segments and signed checkpoints;
- object existence/integrity verification;
- recovery-point enumeration;
- restart reconciliation; and
- post-failure verification epochs.

The initial object data path should stay conceptually simple:

```text
receive
    -> SHA-256 while streaming
    -> write temporary object
    -> complete/hash validate
    -> synchronous durability boundary
    -> atomic publish into SHA-256 namespace
    -> verify
    -> reference from immutable manifest
```

Recovery-point publication must remain separate and Journal-gated.

## Failure tests

- disconnect halfway through an object;
- kill the Repository process at each durability boundary;
- kill the Journal process at each Journal durability boundary;
- abrupt host power loss at deliberately selected boundaries;
- resend the same object;
- same object ID with different bytes;
- same Record v1 ID with same bytes and with different bytes;
- submit a manifest referencing a missing object;
- corrupt a committed object;
- corrupt a Journal entry/checkpoint;
- make the Journal unavailable during backup and during a gated operation;
- exhaust Repository storage;
- exhaust Journal storage;
- restart after a clean shutdown;
- restart after an unclean shutdown; and
- re-enumerate and fully reverify retained recovery points after the unclean event.

## Exit gate

Guidon can accept test objects over mTLS, durably publish them, create and journal authoritative records, create a recovery point only after all commit requirements are satisfied, independently detect corruption, survive/reconcile deliberate crashes, and refuse false-success publication when required data or Journal attestation is incomplete.

---

# Phase 2 — First Windows recovery path

**Target:** 4–6 weeks

This phase proves the product idea end to end.

## Implement

- Windows service skeleton;
- domain-joined gMSA execution;
- stable Guidon endpoint UUID;
- unique endpoint mTLS identity and certificate binding;
- protected-source identity;
- explicit filesystem scope;
- file enumeration;
- content capture;
- minimum supported NTFS metadata;
- Repository transfer;
- manifest construction;
- recovery-point commit;
- recovery-point browsing;
- alternate-location file restore;
- original-location file restore;
- restored-content verification;
- factual attribution records; and
- PCAP/Wireshark acceptance tests for the implemented Guidon network path.

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
    -> inspect Repository + Journal records
```

Repeat with interrupted transfer, Repository restart, Journal restart, and an unclean/power-loss verification-epoch test.

## Exit gate

Guidon can truthfully demonstrate:

> A Windows file was captured, durably stored, verified, committed into a specific recovery point, destroyed, recovered, and verified against that recovery point, with the participating identities and authoritative record history preserved.

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
- constrained signed Job v1 objects;
- expiration, nonce, and replay protection;
- endpoint/job binding;
- user identity attribution;
- gMSA execution attribution;
- AD Authorization Broker under dedicated gMSA;
- SID-based authorization;
- short-lived Job-bound AD Authorization Assertion v1;
- fail-closed authorization behavior;
- denied/not-determined operation records;
- certificate renewal/revocation behavior; and
- minimal operational UI or CLI needed to exercise the system safely.

## Exit gate

An authorized administrator can start a constrained backup or restore operation without granting Guidon arbitrary remote-command capability, and Guidon can show who requested the operation, what user/gMSA/transport identities participated, what authorization source was queried, and what actually occurred.

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
- post-recovery verification with exact stated checks.

## Exit gate

Destroy or replace a protected Windows data volume and reconstruct it from Guidon with supported content and metadata verified.

---

# Phase 5 — Windows bare-metal disaster recovery

**Target:** 10–16 weeks

This is the largest Windows engineering phase and a major product milestone.

## Implement

- Guidon recovery environment;
- offline Recovery Authority trust path;
- short-lived scoped recovery certificate/credential with proof of possession;
- signed Recovery Job;
- repository authentication from recovery environment;
- disk and partition reconstruction;
- EFI/system/boot volume recovery;
- Windows system-volume recovery;
- BCD/boot reconstruction as required;
- data-volume integration;
- first-boot validation;
- workload/service validation; and
- recovery-record integration.

## Broken-domain recovery requirement

Bare-metal recovery must not depend on:

- working AD trust;
- knowing the restored local Administrator password;
- a matching historic LAPS password;
- the normal gMSA being immediately usable; or
- successful secure-channel repair.

Secure-channel repair may be attempted but is not the required recovery mechanism.

Implement the controlled first-boot recovery bootstrap:

- signed recovery authorization;
- target/recovery-point/action binding;
- expiration;
- nonce/replay protection;
- one-time temporary local recovery administrator;
- unique random credential;
- tightly controlled credential release;
- TTL/watchdog cleanup;
- account removal; and
- verification that the account is absent and bootstrap disarmed before final validation.

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

# Phase 6 — PostgreSQL

**Target:** 10–14 weeks

PostgreSQL requires both physical and logical protection paths because they solve different recovery problems.

## Physical protection

- cluster discovery and observed system identifier;
- physical base backup;
- WAL archive/stream handling;
- WAL continuity tracking;
- timeline handling;
- point-in-time recovery;
- alternate recovery location; and
- recovered-cluster validation with explicitly defined checks.

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

---

# Phase 7 — Microsoft SQL Server

**Target:** 10–14 weeks

Guidon should orchestrate Microsoft SQL Server's supported native mechanisms rather than reimplementing database internals.

## Implement

- instance/database discovery;
- native identifier capture where observed;
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
- restore validation with explicitly defined checks;
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

Expected cumulative target for Windows + PostgreSQL + MSSQL capability: **approximately 12–17 months.**

---

# Phase 8 — Guidon and repository disaster recovery

**Target:** 6–10 weeks

The backup system must itself be recoverable.

## Test and implement recovery from

- controller database loss;
- controller host loss;
- Repository service loss;
- Journal service loss;
- Repository host replacement;
- destroyed indexes/catalogs;
- signing-key generation changes; and
- software version changes.

## Required capabilities

- import/mount intact repository data;
- enumerate self-describing recovery points;
- rebuild catalog/index data from repository authority;
- verify manifests and referenced objects;
- verify Journal receipts/checkpoints and expose gaps/mismatches;
- distinguish unavailable data from integrity mismatch;
- preserve historical record continuity without manufacturing missing history; and
- recover protected workloads without requiring the original controller database.

Repository replication or external Journal anchoring may be introduced here if earlier testing justifies them.

## Exit gate

Destroy the Guidon controller/catalog, rebuild Guidon around intact repository data, rediscover recovery points, reverify them, reconcile Journal history, and perform a real workload recovery.

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
- retention implementation;
- garbage collection;
- verification/scrubbing scheduling;
- post-storage-event reverification;
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
- PCAP/Wireshark regression acceptance for every implemented Guidon-controlled connection type;
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
- PostgreSQL clusters and logical objects;
- SQL Server databases;
- Guidon controller/catalog state;
- Repository service infrastructure; and
- Journal service infrastructure.

Include deliberate crash/power-loss tests followed by full retained-data reverification so integrity problems are discovered at the event, not months later during an emergency restore.

## Exit gate

A pilot organization can install Guidon, operate supported protection jobs, understand failures, perform documented recoveries, and independently confirm what Guidon claims about those recoveries.

A reasonable target is **approximately 18–24 months for a tightly controlled pilot candidate** and **24–30 months for broader external pilot readiness**, assuming the scope remains disciplined and no major platform blocker forces redesign.

---

# Scope discipline

The current active workload boundary is:

```text
Windows Server / Workstation
PostgreSQL
Microsoft SQL Server
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
store exact bytes
    -> prove current integrity
    -> recover a file
    -> recover a volume
    -> recover a machine
    -> recover the machine when surrounding identity infrastructure is damaged
    -> recover PostgreSQL to a point in time and at logical granularity
    -> recover SQL Server to a point in time
    -> recover Guidon itself
    -> prove the whole system under sustained failure and scale
```

Guidon never advances a capability because the backup side merely appears to work.

**Recovery remains the mission.**
