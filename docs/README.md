# Guidon Engineering Documentation

This directory contains Guidon's current pre-alpha architecture and implementation contracts.

These documents intentionally separate **what must remain true** from implementation details that may change as code and testing provide evidence.

Guidon follows one overriding reporting rule:

> **Guidon reports only what it can identify, observe, authenticate, resolve, receive, store, or cryptographically verify. It does not turn incomplete facts into administrator conclusions.**

## Architecture

- [Repository](architecture/REPOSITORY.md) — objects, manifests, recovery points, durability, ZFS requirements, crash reconciliation, and post-failure reverification.
- [Journal](architecture/JOURNAL.md) — independent record attestation, streams, segments, receipts, checkpoints, signing, durability, and gating.
- [Identity and trust](architecture/IDENTITY-AND-TRUST.md) — endpoint/user/gMSA identity, PKI, AD authorization, Controller signing, and break-glass Recovery Authority.
- [Network security](architecture/NETWORK-SECURITY.md) — mTLS, identity binding, factual network observations, and PCAP/Wireshark acceptance.
- [At-rest confidentiality](architecture/AT-REST-CONFIDENTIALITY.md) — initial ZFS storage-encryption boundary, offline-media threat model, key separation, and explicit unsupported confidentiality claims.
- [FreeBSD Jail boundaries](architecture/JAIL-BOUNDARIES.md) — Repository/Journal responsibility, filesystem visibility, privilege, and host-root limitations.
- [Threat model](architecture/THREAT-MODEL.md) — prevent/detect/fail-closed/record/recover/outside-claim classifications.

## Contracts

- [Record v1](contracts/RECORD-V1.md) — immutable factual history, provenance, attribution, subjects, correlations, and integrity.
- [Authoritative Record finalization v1](contracts/AUTHORITATIVE-RECORD-FINALIZATION-V1.md) — exact point where Repository authoritative Record bytes become final, including ISS-system clock insertion before hashing/Journal submission.
- [Exact-byte encoding v1](contracts/EXACT-BYTE-ENCODING-V1.md) — UTF-8/no-BOM strict JSON profile, duplicate-key rejection, exact-byte hashing, and parser resource bounds.
- [Signature v1](contracts/SIGNATURE-V1.md) — detached signature envelope, key identity, authority separation, and mandatory Guidon signature-domain separation.
- [Recovery Point Commit v1](contracts/RECOVERY-POINT-COMMIT-V1.md) — manifest/preparation-record/Journal-receipt/publication binding required before `COMMITTED`.
- [Repository Format v1](contracts/REPOSITORY-FORMAT-V1.md) — catalog-independent repository bootstrap, stable repository identity, format compatibility, and portable Journal verification-key history.
- [TLS Profile v1](contracts/TLS-PROFILE-V1.md) — TLS version floor, mTLS, identity binding, revocation behavior, replay-sensitive 0-RTT prohibition, and network acceptance requirements.
- [Verification states](contracts/VERIFICATION-STATES.md) — exact meanings of RECEIVED, STORED, VERIFIED, COMMITTED, RECONSTRUCTION_VERIFIED, RESTORED, and VALIDATED.
- [Job and authorization v1](contracts/JOB-AND-AUTHORIZATION-V1.md) — constrained signed jobs and short-lived AD Authorization Assertions.
- [Retention and deletion](contracts/RETENTION-AND-DELETION.md) — recovery-point deletion, object garbage collection, Journal gating, and safe interruption behavior.
- [Time and configuration provenance](contracts/TIME-AND-CONFIGURATION-PROVENANCE.md) — producer clocks, ISS-system clock comparison, Journal time, immutable configuration generations, and policy provenance.

## Cross-contract precedence

The contracts are intended to be read together. Where a general architecture document describes a concept that has a dedicated v1 implementation contract, the dedicated v1 contract governs the implementation-specific detail while preserving the architecture invariant.

In particular:

- authoritative Record finalization is governed by `AUTHORITATIVE-RECORD-FINALIZATION-V1.md`;
- exact-byte JSON/hash behavior is governed by `EXACT-BYTE-ENCODING-V1.md`;
- detached Guidon signatures and purpose/domain separation are governed by `SIGNATURE-V1.md`;
- recovery-point commit artifact binding is governed by `RECOVERY-POINT-COMMIT-V1.md`;
- catalog-independent repository bootstrap and Journal public-key portability are governed by `REPOSITORY-FORMAT-V1.md`; and
- Guidon v1 TLS implementation behavior is governed by `TLS-PROFILE-V1.md`.

## Status

These contracts are intended to guide implementation and testing. Numeric tuning values, storage layout optimizations, chunking strategy, UI presentation, and other implementation details remain open unless a document explicitly marks them as a requirement.

Phase sequencing belongs in [../ROADMAP.md](../ROADMAP.md), not in these contract documents.
