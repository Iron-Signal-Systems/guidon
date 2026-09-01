# Signature v1

## Purpose

Guidon uses detached signatures for different authority classes. This contract defines a common signature envelope and mandatory domain separation so a valid signature created for one Guidon purpose cannot be silently reinterpreted as authorization for another.

## Governing rules

> **A Guidon signature proves only the exact purpose, key, algorithm, metadata, and payload relationship defined by its signature domain.**

> **Different authority classes use different private keys. Domain separation is additional protection, not a substitute for key separation.**

## Signature envelope

Signature v1 contains exactly these common fields unless a consuming schema explicitly adds separately validated outer metadata:

```text
schema = guidon.signature
schema_version = 1
signature_id = UUIDv7
purpose
algorithm
signing_key_id
payload_sha256
created_at
signature
```

The JSON envelope follows `EXACT-BYTE-ENCODING-V1.md`.

`created_at` is an RFC 3339 timestamp with explicit UTC `Z` offset. Its exact serialized string is part of the signed statement.

## Payload binding

`payload_sha256` is:

```text
sha256:<64 lowercase hexadecimal characters>
```

calculated over the exact payload bytes.

The payload bytes are not reserialized before hashing.

## Signed statement

The Ed25519 signature is calculated over a deterministic UTF-8 statement that binds all authority-relevant Signature v1 metadata except the signature bytes themselves.

The exact v1 signing input is:

```text
GUIDON-SIGNATURE-V1\n
signature_id=<UUIDv7 canonical lowercase string>\n
purpose=<controlled purpose>\n
algorithm=<controlled algorithm>\n
signing_key_id=<sha256:lowercase hex>\n
payload_sha256=<sha256:lowercase hex>\n
created_at=<exact validated RFC3339 UTC string>\n
```

Rules:

- UTF-8 without BOM;
- ASCII field names exactly as shown;
- LF (`0x0A`) separators only;
- no spaces before/after `=`;
- no additional lines;
- final LF is included; and
- values must already have passed their schema validation before the signing input is constructed.

Changing `signature_id`, `purpose`, `algorithm`, `signing_key_id`, `payload_sha256`, or `created_at` therefore invalidates the signature.

## Purpose/domain values

Initial controlled purposes include:

```text
GUIDON:JOURNAL-RECEIPT:V1
GUIDON:JOURNAL-CHECKPOINT:V1
GUIDON:JOURNAL-KEY-TRANSITION:V1
GUIDON:CONTROLLER-JOB:V1
GUIDON:AD-AUTHORIZATION-ASSERTION:V1
GUIDON:RECOVERY-JOB:V1
GUIDON:RECOVERY-CREDENTIAL:V1
```

A new signature purpose requires an explicit contract/schema addition. Implementations must not accept an arbitrary caller-supplied purpose string as authority.

Repository Format v1 does not require a repository-format signing private key. Its bootstrap descriptor uses exact-byte SHA-256 plus durable provenance/trust history rather than inventing another online signing authority.

## Algorithms

The following initial v1 application-signature purposes use Ed25519:

```text
GUIDON:JOURNAL-RECEIPT:V1
GUIDON:JOURNAL-CHECKPOINT:V1
GUIDON:JOURNAL-KEY-TRANSITION:V1
GUIDON:CONTROLLER-JOB:V1
GUIDON:AD-AUTHORIZATION-ASSERTION:V1
GUIDON:RECOVERY-JOB:V1
GUIDON:RECOVERY-CREDENTIAL:V1
```

For these purposes:

```text
algorithm = ed25519
```

Algorithm substitution is prohibited. A verifier does not infer the algorithm from key length and does not accept a different algorithm because the signature would otherwise verify under another library path.

A future algorithm requires a new explicitly permitted purpose/profile version or contract amendment with migration rules.

## Relationship to cryptographic deployment profiles

Signature v1 intentionally remains strict: the initial v1 purposes require `algorithm = ed25519`, and a verifier never substitutes another algorithm because a deployment profile or library supports it.

`../architecture/CRYPTOGRAPHIC-ARCHITECTURE.md` governs the higher-level migration model. A regulated deployment profile may declare a Signature v1 operation unavailable if the required algorithm cannot be executed through an implementation/provider that satisfies that deployment's cryptographic requirements. The operation fails closed rather than silently changing algorithms.

A future approved classical, hybrid, or post-quantum signature uses an explicitly versioned signature/profile/contract transition with its own encoding, key identity, verification, and migration rules. Historical Signature v1 artifacts remain associated with Ed25519 and their original key generations and are not rewritten merely because the active profile changes.

## Signing key identity

For Ed25519 v1:

```text
signing_key_id = sha256:<lowercase SHA-256 of exact raw 32-byte public key>
```

The verifier independently calculates this value from the exact verification public key.

A display name, certificate subject, filename, or configuration alias is not cryptographic key identity.

## Signature encoding

For Signature v1 Ed25519:

```text
signature = 128 lowercase hexadecimal characters
```

representing the exact 64-byte Ed25519 signature.

No base64/base64url autodetection is permitted in v1.

## Verification order

A verifier performs at least:

1. strict Signature v1 parsing under Exact-Byte Encoding v1;
2. supported schema/version check;
3. controlled purpose check;
4. required `algorithm = ed25519` check for the initial v1 purposes;
5. trusted/authorized verification-key lookup for that authority and relevant context;
6. independent `signing_key_id` calculation;
7. exact payload SHA-256 calculation;
8. comparison with `payload_sha256`;
9. exact signing-input construction;
10. Ed25519 signature verification; and
11. artifact-specific authorization, scope, expiry, replay, trust-generation, and policy checks.

A cryptographically valid signature is not automatically authorization. Job, AD authorization, Recovery Authority, Journal, and trust-policy contracts retain separate semantics.

## No generic signing oracle

Guidon components must not expose a generic operation such as:

```text
Sign(bytes)
SignHash(hash)
SignAnything(payload, purpose)
```

Signing entry points are constrained to defined semantics such as:

```text
SignJournalReceipt
SignJournalCheckpoint
SignJournalKeyTransition
SignControllerJob
SignADAuthorizationAssertion
SignRecoveryJob
SignRecoveryCredential
```

The implementation selects the permitted purpose internally for the operation. A remote caller does not choose an unrestricted purpose string.

## Key separation

At minimum:

```text
Journal attestation key
    != Controller Job signing key
    != AD Authorization Broker signing key
    != Recovery Authority key
    != transport mTLS private keys
```

Private-key reuse across independent authority classes is prohibited even with domain separation.

## Key rotation and historical verification

Rotation creates a new `signing_key_id`. Historical signatures remain associated with the exact public key generation that created them.

Key loss, suspected compromise, confirmed compromise, expiration, revocation, and normal rotation are distinct conditions.

Guidon does not rewrite historical signatures during key rotation.

## Acceptance tests

At minimum test:

- correct signature/purpose/payload;
- same signature with changed `purpose`;
- changed `created_at`;
- changed `signature_id`;
- same payload signed by unauthorized authority class;
- altered payload bytes with semantically equivalent JSON;
- altered `payload_sha256`;
- wrong `signing_key_id`;
- wrong algorithm declaration;
- malformed signature length/hex encoding;
- uppercase/alternate encodings rejected where lowercase is required;
- unknown purpose;
- old valid key generation where historical verification remains permitted; and
- revoked/compromised/untrusted key generation according to the consuming policy.
