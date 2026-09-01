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
- [FreeBSD Jail boundaries](architecture/JAIL-BOUNDARIES.md) — Repository/Journal responsibility, filesystem visibility, privilege, and host-root limitations.
- [Threat model](architecture/THREAT-MODEL.md) — prevent/detect/fail-closed/record/recover/outside-claim classifications.

## Contracts

- [Record v1](contracts/RECORD-V1.md) — immutable factual history, provenance, attribution, subjects, correlations, and integrity.
- [Verification states](contracts/VERIFICATION-STATES.md) — exact meanings of RECEIVED, STORED, VERIFIED, COMMITTED, RECONSTRUCTION_VERIFIED, RESTORED, and VALIDATED.
- [Job and authorization v1](contracts/JOB-AND-AUTHORIZATION-V1.md) — constrained signed jobs and short-lived AD Authorization Assertions.
- [Retention and deletion](contracts/RETENTION-AND-DELETION.md) — recovery-point deletion, object garbage collection, Journal gating, and safe interruption behavior.
- [Time and configuration provenance](contracts/TIME-AND-CONFIGURATION-PROVENANCE.md) — producer clocks, ISS-system clock comparison, Journal time, immutable configuration generations, and policy provenance.

## Status

These contracts are intended to guide implementation and testing. Numeric tuning values, storage layout optimizations, chunking strategy, UI presentation, and other implementation details remain open unless a document explicitly marks them as a requirement.

Phase sequencing belongs in [../ROADMAP.md](../ROADMAP.md), not in these contract documents.
