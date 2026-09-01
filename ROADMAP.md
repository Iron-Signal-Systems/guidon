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
- FreeBSD Repository/Journal Jail responsibility boundaries;
- encrypted-at-rest Job storage boundary and key-separation requirements;
- interactive privileged-operation MFA contract, including TOTP replay handling and exact-Job binding;
- mature multi-witness Journal direction without making Phase 1 depend on distributed quorum;
- external write-once advancing witness semantics and explicit host-root limitations;
- Recovery Copy appliance trust boundary, push-only replication direction, and separate recovery-export trust root; and
- initial threat model and explicit unsupported claims.

The detailed contracts are maintained under [`docs/`](docs/README.md).

## Explicitly defer from Phase 1 implementation

The following are architecturally documented now but are not required to be implemented during Phase 1:

- second operational Journal witness on an independent host;
- external COW/write-once advancing witness appliance;
- hardware-backed witness high-water mark or signing key;
- independent Recovery Copy appliance;
- repository replication;
- recovery-export packaging and RSA-4096 Recovery Copy administrator PKI;
- multi-person approval policy;
- high availability;
- chunk-size/chunking optimization beyond the required object contract;
- polished UI;
- advanced retention-policy features beyond the deletion-safety contract;
- mass deployment tooling;
- broad workload abstractions;
- speculative plugin/framework architecture; and
- other hardware-backed/external trust features unless implementation evidence justifies them early.

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
12. How are stored Job artifacts protected from offline disclosure, and what key boundaries protect them?
13. Which interactive privileged operations require MFA, how is the TOTP result bound to the exact Job, and what is recorded without storing the OTP itself?
14. How can future Journal redundancy improve availability without changing the fail-closed meaning of Journal gating?
15. What is the role of the external advancing witness, and what does it not protect against?
16. What trust boundary will a future Recovery Copy appliance enforce between replication ingress and administrator recovery export?

---

# Phase 1 — Repository core

**Target:** 3–5 weeks

Build the smallest correct FreeBSD Repository and Journal path. Phase 1 intentionally implements one Repository and one operational Journal while preserving the interfaces and artifact identities needed for future independent witnesses and Recovery Copy replication.

## Implement

- FreeBSD appliance-host baseline;
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
- stable Journal identity and signing-key identifiers suitable for later multi-witness correlation and external anchoring;
- object existence/integrity verification;
- recovery-point enumeration;
- restart reconciliation;
- post-failure verification epochs;
- encrypted-at-rest persistence for any Phase 1 Job/control artifact that is durably queued or spooled;
- component-specific Job-storage key separation and no plaintext durable Job copies;
- no plaintext Job bodies in normal logs, crash artifacts, or support bundles; and
- Record/authorization fields capable of representing MFA method/result without ever storing the OTP value.

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

Phase 1 does **not** require a second Journal, external witness, Recovery Copy appliance, quorum protocol, active-active Repository, or remote replication implementation.

## Phase 1 forward-compatibility invariants

Implementation must not prevent later addition of:

```text
same exact Record v1 bytes
    -> Journal A
    -> Journal B

sealed Journal checkpoint
    -> external advancing witness

immutable Repository artifacts
    -> Recovery Copy appliance push-only ingress
```

Journal sequence numbers and checkpoint boundaries remain local to each Journal witness. Future witnesses correlate authoritative records by `record_id` and exact `record_sha256`, not by requiring identical internal Journal streams.

Normal `COMMITTED` remains the local authoritative Repository recovery-point state. Future replication status is a separate protection property and does not redefine `COMMITTED`.

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
- restart after an unclean shutdown;
- re-enumerate and fully reverify retained recovery points after the unclean event;
- inspect durable Phase 1 Job/control storage and confirm no plaintext Job body is present;
- copy Phase 1 persistent storage offline and confirm Job ciphertext cannot be decrypted without the correct component key boundary;
- verify a Job decrypts to the exact originally signed bytes before signature validation;
- verify wrong-key, altered-ciphertext, altered-tag, and truncated encrypted Job artifacts fail closed; and
- verify logs/support output never records an entered OTP or durable plaintext Job body.

## Exit gate

Guidon can accept test objects over mTLS, durably publish them, create and journal authoritative records, create a recovery point only after all commit requirements are satisfied, independently detect corruption, survive/reconcile deliberate crashes, refuse false-success publication when required data or Journal attestation is incomplete, and preserve the Phase 0 confidentiality/authorization boundaries needed for later signed encrypted Jobs and multi-system recovery architecture.

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
    -> inspect Repository + Journal history
```

## Exit gate

Guidon can protect and recover a Windows file end to end while preserving the defined identity, authorization, transport, integrity, durability, and factual-history requirements.

---

# Phase 3 — Windows operational protection and control

**Target:** 4–8 weeks

Expand the Windows path into repeatable operational protection without introducing arbitrary remote administration.

## Implement

- repeatable file/directory backup jobs;
- constrained signed Job v1 execution;
- encrypted-at-rest Job envelopes for Controller and endpoint durable queues/spools;
- decrypt-to-exact-signed-bytes verification path with plaintext confined to controlled memory;
- AD Authorization Assertions for applicable manual privileged operations;
- TOTP-based MFA v1 for policy-defined interactive privileged operations;
- default TOTP timestep of 30 seconds with bounded skew policy and replay prevention;
- MFA authorization binding to exact `job_sha256` rather than a broad privileged session;
- scheduled/system operations with truthful `user.presence = not_present` attribution and no interactive MFA requirement;
- retention/deletion policy generation references;
- safe recovery-point deletion and object-GC flow;
- configuration/policy generation activation and provenance;
- source/repository clock comparison and recorded time provenance;
- endpoint certificate rotation/rebinding flow;
- service recovery/restart behavior;
- operator-facing exact failure states;
- basic operational status surfaces; and
- expanded security/failure tests.

## Exit gate

Guidon can protect a practical Windows file/directory scope repeatedly, perform controlled recovery/deletion operations under the defined signed-Job, encrypted-at-rest, authorization, and MFA model, and explain exactly who/what performed each meaningful action without retaining OTP secrets.

---

# Phase 4 — Windows volume recovery

**Target:** 6–10 weeks

Implement a defined Windows volume-level protection/recovery path.

## Implement

- supported Windows volume capture model;
- consistency/VSS boundary;
- volume reconstruction manifest;
- required boot/filesystem metadata where applicable;
- alternate-target volume reconstruction;
- integrity verification after reconstruction;
- original-target replacement flow under explicit authorization;
- recovery-space/capacity checks; and
- interruption/restart behavior.

## Exit gate

Guidon can reconstruct and verify a supported Windows data volume from committed recovery data.

---

# Phase 5 — Windows bare-metal DR

**Target:** 10–16 weeks

Recover a failed Windows system without making successful disaster recovery depend on the original AD trust or unknown historical local credentials.

## Implement

- bootable Guidon recovery environment;
- full-system reconstruction path;
- storage/partition/boot reconstruction;
- system/boot-volume handling;
- protected metadata required for supported recovery;
- offline Recovery Authority verification;
- signed/scoped/short-lived recovery authorization;
- independent recovery MFA that does not depend on production AD availability;
- encrypted-at-rest recovery Jobs and separately protected temporary recovery credentials;
- one-time temporary recovery-administrator bootstrap;
- one-time credential display/retrieval process;
- expiration/watchdog cleanup;
- secure-channel repair as an optional convenience attempt;
- domain/gMSA re-establishment where possible;
- recovery-bootstrap disarm/removal verification;
- full factual recovery history; and
- workload/system validation checks.

## Failure/abuse tests

- AD unavailable;
- secure channel broken;
- old machine password unusable;
- original local Administrator password unknown;
- historical LAPS password unavailable/invalid;
- replay old recovery authorization;
- replay a previously accepted TOTP authorization against another Job;
- use recovery authorization for the wrong endpoint;
- interrupt recovery before first boot;
- interrupt recovery after temporary administrator creation;
- expire temporary recovery access;
- fail domain repair and continue controlled local recovery;
- verify bootstrap cleanup before final `VALIDATED` state.

## Exit gate

Guidon can rebuild a supported Windows system from recovery data and regain controlled administrative access without depending on the failed environment's AD trust, an unknown historic local password, or production-AD availability for recovery MFA.

---

# Phase 6 — PostgreSQL

**Target:** 10–14 weeks

Implement PostgreSQL protection and recovery with clear physical/logical semantics.

## Implement

- PostgreSQL identity/provenance model;
- physical base backup path;
- WAL capture/continuity;
- point-in-time recovery;
- logical database/schema/table backup and restore paths where supported;
- alternate-location/isolated recovery;
- consistency verification;
- reconstruction verification;
- workload-specific validation; and
- recovery interruption/reconciliation behavior.

## Exit gate

Guidon can perform and verify supported PostgreSQL recovery paths, including a demonstrated point-in-time recovery.

---

# Phase 7 — Microsoft SQL Server

**Target:** 10–14 weeks

Implement SQL Server protection and recovery with native backup semantics.

## Implement

- SQL Server instance/database identity/provenance;
- full backup;
- differential backup;
- transaction-log backup;
- complete database recovery;
- point-in-time recovery;
- alternate-name/location recovery;
- recovery-chain validation;
- isolated restore path;
- workload-specific validation; and
- later granular extraction through supported isolated restore mechanisms rather than an unsafe custom database parser.

## Exit gate

Guidon can perform and verify supported MSSQL full/differential/log recovery chains and point-in-time recovery.

---

# Phase 8 — Guidon and repository DR

**Target:** 6–10 weeks

Prove that Guidon's own control plane and mutable catalog are not hidden single points of failure, and implement the independent recovery-copy path without turning the secondary system into another active Repository.

## Implement

- clean Guidon rebuild procedure;
- repository-format discovery/import;
- catalog/index reconstruction from authoritative repository artifacts;
- Journal history/receipt/checkpoint verification;
- signing/public-key history recovery requirements;
- configuration/policy generation recovery;
- missing/corrupt metadata handling;
- recovery when original Controller is unavailable;
- recovery when Journal service must be rebuilt from surviving durable history;
- recovery when repository datasets are moved/imported to a new host;
- documented appliance/storage replacement procedure;
- independent Recovery Copy appliance/system;
- replication ingress that accepts only authorized Guidon push traffic;
- no normal read/delete/modify/execute capability granted to the primary Guidon through replication credentials;
- independent verification of received objects/manifests/commit artifacts/Journal receipts/public verification material;
- Recovery Copy replication receipts/status;
- separate Recovery Copy administrative/export interface;
- separate Recovery Copy trust root;
- RSA-4096 Recovery Copy administrator identity/certificate profile;
- MFA on recovery-export authorization;
- export of self-contained Guidon recovery material for administrator-controlled transfer/import into a clean Guidon appliance;
- independent verification by the receiving clean Guidon rather than trust-by-source; and
- Recovery Copy-local retention/deletion authority rather than remote deletion by the primary Guidon.

## Exit gate

Starting from intact authoritative Guidon recovery data or an independently verified Recovery Copy export, but without the original Controller/catalog, a clean Guidon installation can rediscover supported recovery points, verify them, rebuild operational indexes, and recover a supported workload. Compromise of the primary Guidon replication credential alone cannot read back, modify, or erase retained Recovery Copy content.

---

# Phase 9 — Pilot hardening

**Target:** 4–8 months

Turn successful engineering recovery paths into something suitable for a controlled real-world pilot.

## Implement / prove

- installer and appliance lifecycle;
- signed release/update mechanism;
- upgrade and rollback strategy;
- supported hardware/storage profiles;
- capacity planning and storage-pressure behavior;
- resource exhaustion/DoS testing;
- fuzzing and malformed-input testing;
- concurrency/race testing;
- certificate/key rotation drills;
- recovery-authority rotation/loss/compromise procedures;
- Journal-key rotation/loss/compromise procedures;
- retention/GC abuse testing;
- longer-running backup/recovery tests;
- representative scale testing;
- operator documentation/runbooks;
- alerting/health/status surfaces;
- support bundle/diagnostic export without secret leakage;
- backup/recovery acceptance evidence for each supported workload;
- independent security/code review;
- pilot deployment/rollback procedure;
- optional second operational Journal witness on an independent host, with normal gating policy able to operate at 1-of-2 while reporting degraded redundancy;
- cross-Journal conflict detection using common `record_id` + exact `record_sha256`;
- optional external COW/write-once advancing witness that asynchronously anchors sealed Journal checkpoints;
- no external-witness dependency for ordinary operational availability;
- external-witness rollback/high-water-mark protection design, with hardware backing where justified; and
- appliance health surfaces that expose Journal witness count, degraded attestation, external-anchor lag, and Recovery Copy protection status as distinct facts.

## Pilot exit gate

Guidon can be installed, operated, upgraded, intentionally broken, recovered, audited, and removed in a controlled pilot without relying on undocumented developer knowledge. Where optional independent witnesses/Recovery Copy are deployed, the pilot demonstrates their claimed failure-domain and authorization properties rather than treating their presence as proof by configuration alone.

---

# Mature multi-system trust direction

The intended mature architecture is deliberately asymmetric:

```text
SYSTEM 1 — Guidon Recovery Appliance
    Repository
    Journal A

SYSTEM 2 — Guidon Witness Appliance
    Journal B

SYSTEM 3 — External Witness
    isolated COW / write-once / always-advancing checkpoint anchor

SYSTEM 4 — Recovery Copy Appliance
    receive-only from authorized Guidon replication path
    independent immutable recovery storage
    separate administrator recovery/export trust root
```

Roles remain distinct:

```text
Repository
    owns recovery data and recovery meaning

Journal A / Journal B
    independently attest exact authoritative Record v1 bytes
    support operational Journal gating

External Witness
    anchors sealed Journal history outside the operational trust/failure domain
    does not participate in normal customer recovery
    does not normally gate backup availability

Recovery Copy
    preserves independent recovery material
    accepts push-only replication from Guidon
    does not expose normal read/delete/modify authority to the primary
    exports recovery material only through the separate recovery-administration path
```

Normal future Journal redundancy should prefer:

```text
configured operational witnesses = 2
normal required witnesses = 1
healthy desired witnesses = 2
```

A single available authorized Journal can allow ordinary gated operation while Guidon reports degraded witness redundancy. Zero available required witnesses still fails closed.

The external witness is append-only at the application/protocol level: prior anchors are never modified, sequence/head only advances, and each anchor cryptographically commits to the prior anchor. Copy-on-write storage supports this design but is not by itself treated as a WORM guarantee. Malicious root on the witness host remains outside software-only protection claims unless an independent rollback-resistant hardware/high-water-mark mechanism establishes otherwise.

---

# What remains intentionally outside the roadmap

The following remain outside the current committed roadmap unless implementation or pilot evidence justifies adding them:

- Kubernetes/container backup;
- VMware/Hyper-V platform-level protection;
- Microsoft 365;
- Exchange;
- Oracle;
- broad cloud-provider coverage;
- generic remote execution;
- arbitrary plugin framework;
- large microservice fleet;
- distributed consensus architecture;
- active-active repository design;
- mandatory external HSM dependencies; and
- feature breadth added primarily for competitive checklists rather than demonstrated recovery need.

---

# Roadmap decision rule

When deciding whether to add work, prefer the option that improves one or more of:

1. recoverability;
2. integrity;
3. explainability;
4. security/authority separation;
5. failure behavior;
6. operational simplicity; or
7. ability to test the recovery claim.

Do not add complexity solely because another product contains the feature.
