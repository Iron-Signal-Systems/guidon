# Cryptographic Architecture and Compliance Profiles

## Purpose

Guidon uses cryptography for transport protection, object and record integrity, signatures, authorization, Job/control confidentiality, storage confidentiality, MFA-secret protection, recovery authority, and independent witness/recovery-copy trust.

This document freezes the architectural rules that keep those uses strong, separable, truthful, and replaceable as standards and regulatory requirements evolve.

The governing rule is:

> **Compliance requirements establish a minimum acceptable cryptographic floor. They do not establish Guidon's maximum security level.**

Guidon may use stronger approved algorithms, parameters, key-protection mechanisms, or assurance boundaries when they are operationally appropriate and compatible with the applicable deployment profile.

## Validated implementation, not algorithm-name compliance

Guidon distinguishes:

```text
approved algorithm
    !=
validated cryptographic module
    !=
compliant product/deployment
```

Using AES, SHA-2, RSA, ECDSA, EdDSA, ML-KEM, ML-DSA, or another approved algorithm does not by itself establish FIPS 140-3 validation or CJIS compliance.

Where a deployment policy requires a validated cryptographic module, the exact cryptographic provider/module, version, operating environment, validation status/certificate, enabled mode, and Guidon build/runtime relationship must satisfy the applicable requirement.

Guidon must not describe itself or a deployment as FIPS validated/certified merely because:

- an algorithm is FIPS-approved;
- a cryptographic library has a FIPS-capable mode;
- another version of the same library/module has a validation certificate;
- a validation is planned, in process, or pending; or
- a storage product advertises AES encryption without an applicable validated-module claim.

Compliance claims are deployment/profile facts and remain bounded by what can actually be established.

## FIPS 140-3 and CJIS direction

Guidon is architected so deployments that require FIPS 140-3/CJIS-compatible cryptographic implementation can select an appropriate regulated profile without changing Guidon's Repository object model, authoritative Record model, Recovery Point identity, or recovery semantics.

The intended direction is:

```text
Guidon security architecture
    -> strong default cryptographic profile
    -> optional/required regulated profile
        -> approved algorithms and parameters
        -> validated cryptographic implementation where required
        -> exact module/provider provenance
        -> fail closed when the required profile cannot be established
```

For CJI deployments, Guidon treats the current CJIS cryptographic requirements as a minimum baseline and may impose stronger requirements where Guidon policy or the customer requires them.

A future regulatory change must not require silently weakening an existing Guidon security property merely to preserve compatibility.

## Cryptographic profile model

Cryptographic policy is explicit and versioned.

Conceptually, a Guidon cryptographic profile identifies at least:

```text
profile_id
profile_version
profile_generation
purpose / applicable component classes
allowed algorithms
minimum parameters / key strengths
required provider/module class
required validation state where applicable
required key-protection class where applicable
transition / deprecation state
configuration provenance
```

The exact serialized schema is frozen when implementation requires it.

A cryptographic profile is separate from an artifact schema version.

For example:

```text
Record v1
    !=
Signature v1
    !=
TLS Profile v1
    !=
cryptographic deployment profile generation
```

Changing a deployment cryptographic profile does not silently reinterpret historical artifacts.

## No silent downgrade

Guidon fails closed rather than silently falling back to a weaker or unapproved algorithm/provider when a configured profile requires a stronger one.

Examples include:

```text
required validated provider unavailable
    -> fail closed for operations requiring that provider

required signature algorithm unavailable
    -> do not substitute another signature algorithm

required TLS profile cannot be negotiated
    -> connection rejected

required storage-encryption assurance cannot be established
    -> do not report the storage as satisfying that assurance profile
```

Compatibility behavior, where intentionally permitted, must be explicit in policy/configuration provenance and observable in Guidon records/status.

## Key-purpose separation

Cryptographic agility does not weaken Guidon's existing key-separation requirement.

Independent authority classes remain independently keyed, including at minimum:

```text
endpoint transport identity
infrastructure transport identity
Controller Job signing
Journal attestation signing
AD Authorization Broker signing
Recovery Authority
Recovery Copy administration
Job-storage encryption
MFA/TOTP secret protection
Repository/storage encryption
Recovery Copy storage encryption
future witness/high-water-mark authority
```

One algorithm/provider may implement several approved operations, but private keys and authority scopes do not collapse merely because the implementation provider is shared.

## Algorithm and key-generation identity

Every cryptographic artifact whose future interpretation depends on an algorithm or key generation must carry or unambiguously reference enough information to determine:

```text
algorithm/profile
key identity
generation/version
purpose/domain
relevant parameters
```

A verifier must not infer a different algorithm from key length, signature length, certificate subject, filename, or library autodetection.

Public verification material is stored in the exact canonical representation defined by its declared signature/profile contract. Guidon does not pad, truncate, widen, or reinterpret native key material merely to fit a universal fixed-width field.

Where an existing v1 contract deliberately fixes one algorithm, that v1 contract remains strict. A future algorithm uses an explicitly versioned profile/schema/contract transition rather than opportunistic substitution.

## Historical verification survives migration

Cryptographic migration does not rewrite historical truth.

Guidon retains the public verification material and profile metadata needed to interpret historical signatures, receipts, checkpoints, certificates, and other cryptographic facts for as long as policy requires those artifacts to remain verifiable.

Migration rules include:

- a new private key creates a new key identity/generation;
- a new signature algorithm/profile does not cause historical artifacts to be re-signed and presented as original facts;
- deprecated algorithms may remain verification-only when policy permits;
- known/suspected compromise is recorded separately from normal deprecation or key loss;
- trust/profile transitions are authoritative security events and are Journal-gated where applicable; and
- inability to verify an old artifact under current trust material remains visible rather than being silently ignored.

## Post-quantum migration direction

Guidon does not assume today's public-key algorithms remain the permanent choice for long-lived recovery history.

NIST-standardized post-quantum algorithms and future validated implementations may be introduced through new explicit cryptographic profiles/contracts.

The architecture must permit independent migration of:

```text
transport key establishment
transport authentication/signatures
Journal attestation signatures
Controller Job signatures
AD Authorization Broker signatures
Recovery Authority credentials/signatures
Recovery Copy administration credentials/signatures
external-witness/high-water-mark signatures
```

Guidon may use classical, hybrid, or post-quantum profiles when approved by the applicable standards and deployment policy.

No Phase 1 component is required to implement post-quantum cryptography merely to satisfy this architectural invariant.

## Repository object identity remains independent

Repository Format v1 identifies immutable recovery-object bytes using SHA-256 according to the Repository contract.

That object-identity rule is separate from:

- storage encryption;
- transport encryption;
- Job encryption;
- detached signatures; and
- future post-quantum public-key migration.

A change to a storage-encryption provider or cryptographic module must not require rewriting Repository object identities merely because the encryption boundary changed.

If Guidon later changes the object-identity hash contract, that requires an explicit Repository Format version/migration contract rather than being implied by a cryptographic deployment profile.

## At-rest regulated profile boundary

ZFS native encryption is an implementation/storage confidentiality mechanism; it is not automatically a FIPS 140-3 or CJIS compliance claim.

A deployment that requires a validated at-rest cryptographic boundary must establish an allowed implementation whose exact module/provider/operating environment satisfies the applicable profile.

This may be implemented below, within, or above the Repository logical object layer provided that:

- Repository logical object identity remains stable unless an explicit Repository Format migration says otherwise;
- the required recovery keys/material are included in disaster-recovery planning;
- the encryption mechanism does not make catalog-independent recovery impossible without an explicitly documented dependency; and
- Guidon truthfully reports what at-rest assurance it actually established.

Phase 0 does not choose the final regulated storage-encryption implementation.

## Implementation boundary

Guidon implementation should centralize cryptographic operations behind narrow purpose-specific code boundaries rather than scattering unrestricted crypto-library calls throughout business logic.

Examples:

```text
SignJournalReceipt
VerifyJournalReceipt
EncryptDurableJob
DecryptDurableJob
EstablishGuidonTLS
VerifyRecoveryAuthorization
```

This is not a generic cryptographic oracle. Callers provide operation-specific inputs; the cryptographic boundary selects the permitted algorithm/profile for that purpose.

A regulated build/deployment should be capable of establishing at startup/runtime:

- the selected cryptographic profile;
- whether the required cryptographic module/provider initialized correctly;
- the module/provider version or equivalent identity;
- whether required self-tests/integrity checks succeeded where the module exposes those facts; and
- whether the configured operation is allowed under the active profile.

Failure to establish a required regulated profile fails closed for operations that require that profile.

## Configuration and audit provenance

Security-relevant cryptographic configuration is versioned configuration provenance.

Where applicable, Guidon records facts such as:

```text
cryptographic profile generation
provider/module identity and version
validation certificate/reference when actually established
FIPS/approved-only mode state when actually established
algorithm/key generation used by an operation
trust generation
configuration activation time
producer/component that observed the state
```

Guidon does not record a compliance assertion merely because a configuration flag requested one.

Requested mode and established mode are different facts.

## Phase 1 requirements

Phase 1 does not need to deliver every future regulated deployment profile.

It must, however, preserve these invariants:

1. Repository/Record/Recovery Point formats do not depend on one permanent public-key algorithm.
2. Signature v1 remains strict and does not permit algorithm substitution.
3. TLS behavior remains profile-driven and fail closed.
4. Job encryption remains algorithm/profile identified and authenticated.
5. Cryptographic keys remain separated by authority/purpose.
6. Cryptographic configuration/provider state can be represented in configuration provenance.
7. A future validated cryptographic provider can replace/augment the initial provider without redesigning recovery semantics.
8. Storage encryption remains separable from Repository logical object identity.
9. Historical verification material is retained across approved key/algorithm transitions.
10. No component labels itself FIPS/CJIS compliant solely from an algorithm name or requested configuration mode.

## Explicit non-claims

Guidon does not claim that:

- FIPS 140-3 validation alone makes a complete Guidon deployment compliant with CJIS or another regulation;
- using a FIPS-approved algorithm is equivalent to using a validated cryptographic module;
- a provider's FIPS mode applies to an operating environment/version not covered by the provider's validation;
- ZFS encryption by itself establishes a regulated cryptographic-module claim;
- post-quantum algorithms are required in Phase 1;
- cryptographic agility means arbitrary runtime algorithm choice; or
- stronger cryptography compensates for compromised authorization, malicious host root, stolen valid credentials, or other failures outside the cryptographic boundary.
