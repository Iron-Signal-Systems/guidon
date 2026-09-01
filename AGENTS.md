# Guidon Agent and Contributor Rules

## Purpose

This file defines how contributors, coding agents, and automation should work within the Guidon repository.

It is an engineering map and behavioral contract. It does not replace the architecture and implementation contracts under `docs/`.

Guidon is built around one governing principle:

> **Lead the way. Recovery is the mission.**

Backup creation alone is not success. Guidon engineering must preserve truthful recovery semantics, explicit security boundaries, and observable failure behavior.

## Product Engineering Principles

### Recovery first

Prefer work that improves:

1. recoverability;
2. integrity;
3. explainability;
4. security and authority separation;
5. failure behavior;
6. operational simplicity; and
7. the ability to test a recovery claim.

Do not add complexity merely because another product contains a feature.

### Report only what Guidon can establish

Guidon must not convert incomplete observations into conclusions.

Preserve distinctions such as:

```text
unknown
not_known
not_observed
not_verified
not_performed
not_present
not_applicable
degraded
unavailable
mismatch
failed
```

Do not infer a positive state from the absence of an error.

Do not collapse meaningful Guidon states into generic success/failure when more precise state is available.

### Simple does not mean vague

Guidon should remain operationally simple while retaining authoritative technical detail.

Presentation depth must not change the meaning of underlying facts.

Presentation depth and operational authority are separate concerns.

### Prefer explicit engineering

Prefer simple, narrow, inspectable implementation over speculative abstractions.

Do not create generic plugin frameworks, arbitrary execution systems, catch-all platform frameworks, generic cryptographic/signing oracles, unnecessary service boundaries, or generalized registries/factories for anticipated future needs.

Implement the requirement that exists now.

## Sources of Truth

Read the applicable documents before changing implementation behavior.

### Phase scope and sequence

`ROADMAP.md` defines current phase scope, sequencing, implementation targets, and exit gates.

Do not implement future roadmap phases merely because their architecture has already been documented.

### Architecture invariants

`docs/architecture/` defines system boundaries, trust relationships, recovery semantics, failure assumptions, and long-lived architectural requirements.

### Implementation contracts

`docs/contracts/` defines strict implementation-specific contracts.

Where a dedicated implementation contract exists, it governs implementation-specific behavior while preserving the broader architecture invariant.

Examples:

```text
docs/architecture/REPOSITORY.md
    -> Repository architecture

docs/contracts/REPOSITORY-FORMAT-V1.md
    -> exact Repository Format v1 behavior

docs/architecture/JOURNAL.md
    -> Journal architecture

docs/contracts/SIGNATURE-V1.md
    -> exact Signature v1 behavior

docs/architecture/CRYPTOGRAPHIC-ARCHITECTURE.md
    -> cryptographic architecture/profile invariants
```

Do not silently reinterpret a strict v1 contract through a more general architecture document.

## Scope Discipline

Work only within the current roadmap phase and current engineering slice unless explicitly directed otherwise.

Do not pre-build future systems merely because they are already architecturally defined.

For example, Phase 1 does not require implementation of:

```text
Journal B
External Witness
Recovery Copy appliance
distributed quorum
active-active Repository
broad workload abstraction
polished UI
generic plugin framework
```

Future compatibility should be preserved where required, but future features should not be implemented early without a concrete current requirement.

## Contract Discipline

Code MUST NOT be changed to silently violate a contract.

A contract MUST NOT be changed merely to make failing code pass.

Architecture and contracts must not be changed merely to make implementation easier.

When implementation conflicts with an existing contract:

1. stop at that boundary;
2. identify the exact conflict;
3. determine whether the implementation is wrong, the contract is wrong, or the requirement was misunderstood;
4. surface the issue explicitly; and
5. resolve the discrepancy intentionally before continuing through that boundary.

Do not silently weaken durability requirements, verification meaning, authorization requirements, cryptographic requirements, Journal gating, authority separation, failure behavior, or recovery semantics.

## Security and Trust Rules

No credential, identity, key, or authority silently becomes universal authority.

Keep separate, as applicable:

```text
human identity
gMSA execution identity
service identity
endpoint identity
transport identity
Controller Job signing authority
Journal attestation authority
AD Authorization Broker authority
MFA verifier
Recovery Authority
Recovery Copy administration authority
Job-storage encryption authority
storage-encryption authority
```

Do not infer cryptographic identity from IP address, MAC address, hostname, VLAN, or interface. Those are observations unless another contract establishes more.

Fail closed where a protected operation requires authority, verification, or trust that could not be established.

Do not manufacture authorization, user presence, verification, or historical continuity.

## State and Failure Discipline

Failure behavior is part of the product.

When implementing a durable, destructive, privileged, recovery-sensitive, or security-sensitive operation, implement and test its failure paths at the same time as its successful path.

Examples include:

```text
short transfer
long transfer
hash mismatch
existing immutable-object conflict
storage full
I/O error
durability failure
process termination
dependency unavailable
Journal unavailable
authorization denied
authorization not_determined
MFA unavailable
certificate mismatch
restart with interrupted artifacts
```

Do not postpone fundamental failure behavior to a later hardening phase.

A failed verification is still a factual verification result.

A partially completed operation must preserve what actually occurred.

Do not erase earlier failure history merely because a later retry succeeds.

## Durability Rules

`write()` success is not durability.

Do not report a durability-dependent Guidon state until the exact required durability boundary has completed.

Do not assume filesystem or storage durability semantics from documentation alone when the architecture requires platform validation.

Durability-sensitive implementation must eventually be tested on the actual supported operating system, filesystem, storage stack, and representative hardware.

Interrupted durable artifacts should be preserved when required for truthful reconciliation.

Cleanup must not destroy information required to determine what actually occurred.

## Immutable Data Rules

Immutable Guidon artifacts are never silently overwritten to resolve a conflict.

For content-addressed Repository objects:

```text
existing identity
    -> independently verify existing bytes
    -> reuse only if they actually match
```

A conflicting object is an integrity condition.

Do not replace the existing object merely because a newly received object claims the same identity.

## Cryptographic Rules

Use the applicable cryptographic contract/profile.

Do not infer an algorithm from key size or signature size, silently downgrade algorithms/providers, treat an approved algorithm as proof of a validated module, treat requested FIPS mode as proof that FIPS mode was established, reuse keys across independent authority classes, or create generic signing/encryption oracles.

Cryptographic operations should remain purpose-specific.

Historical cryptographic artifacts are not rewritten merely because the active cryptographic profile changes.

## Secrets and Sensitive Material

Do not intentionally log or persist:

```text
private keys
OTP values
TOTP shared secrets
plaintext durable Job bodies
temporary recovery credentials
decrypted sensitive control payloads
```

Any exception must be explicitly defined by a security contract.

Support bundles and crash artifacts must not become accidental secret-storage paths.

## Repository Operations

Do not perform repository writes unless explicitly authorized for the specific action.

This includes commit, push, merge, pull-request creation, branch changes, ruleset/settings changes, issue creation, repository-content deletion, and other GitHub/repository writes.

Read-only repository inspection and local/offline working changes are permitted unless explicitly restricted.

Permission to create or modify local working files is not permission to commit or push them.

Permission for one repository write applies only to the specifically approved action/change set and does not carry forward automatically.

## Review Expectations

Before proposing a change as complete:

- compare the implementation against the applicable architecture and contracts;
- verify that failure behavior has not been weakened;
- verify that security boundaries remain explicit;
- verify that no future phase was accidentally pulled forward;
- verify that state reporting remains truthful;
- run all applicable checks defined by nested `AGENTS.md` files; and
- clearly report any check that could not be executed.

## Nested AGENTS.md Files

More specific directories may contain their own `AGENTS.md`.

A nested file may add or refine requirements for that portion of the tree but must not silently weaken repository-wide architecture, security, recovery, or truthfulness requirements.

Current nested engineering standard:

```text
go/AGENTS.md
    -> applies to the complete Go module tree
```
