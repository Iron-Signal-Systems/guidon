# Signature v1

## Purpose

Guidon uses detached signatures for different authority classes. This contract defines a common signature envelope and mandatory domain separation so a valid signature created for one Guidon purpose cannot be silently reinterpreted as authorization for another.

## Governing rules

> **A Guidon signature proves only the exact purpose, key, algorithm, and payload relationship defined by its signature domain.**

> **Different authority classes use different private keys. Domain separation is additional protection, not a substitute for key separation.**

## Signature envelope

Signature v1 contains at least:

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

The exact encoding follows `EXACT-BYTE-ENCODING-V1.md`.

## Payload binding

`payload_sha256` is SHA-256 over the exact payload bytes.

The signature input is not the raw payload alone. It is a deterministic binary/text input that includes an explicit Guidon domain plus the payload SHA-256.

For v1, the logical signing input is:

```text
GUIDON-SIGNATURE-V1\n
<purpose>\n
<payload_sha256 lowercase hex>\n
```

encoded as UTF-8 without BOM and with LF (`0x0A`) separators exactly as shown.

The final LF is included.

The signer signs those exact signing-input bytes using the algorithm specified by the artifact contract.

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
GUIDON:REPOSITORY-FORMAT-DESCRIPTOR:V1
```

New signature purposes require an explicit contract/schema addition. Implementations must not accept an arbitrary caller-supplied purpose string as authority.

## Algorithm

Journal receipts, Journal checkpoints, and Journal key transitions use Ed25519 in v1.

Other authority classes may use Ed25519 or another explicitly frozen algorithm in their own contract, but an implementation must never infer the algorithm from key length or accept algorithm substitution.

`algorithm` is a controlled value and must match the purpose's permitted algorithm.

## Signing key identity

For Ed25519 v1:

```text
signing_key_id = sha256:<lowercase SHA-256 of exact raw 32-byte public key>
```

The verifier independently calculates the key ID from the exact verification public key and compares it with the signed artifact's declared key identity relationship.

A display name, certificate subject, filename, or configuration alias is not cryptographic key identity.

## Verification order

A verifier performs at least:

1. strict Signature v1 parsing;
2. supported schema/version check;
3. controlled purpose check;
4. permitted algorithm check for that purpose;
5. trusted/authorized verification-key lookup for that authority and relevant time/context;
6. independent `signing_key_id` calculation;
7. exact payload SHA-256 calculation;
8. exact signing-input construction;
9. cryptographic signature verification; and
10. artifact-specific authorization/scope/expiry/replay checks.

A cryptographically valid signature is not automatically authorization. Job, AD authorization, Recovery Authority, Journal, and trust-policy contracts retain their separate semantics.

## No generic signing oracle

Guidon components must not expose a generic remote operation such as:

```text
Sign(bytes)
SignHash(hash)
SignAnything(payload, purpose)
```

Signing entry points are constrained to defined semantics such as:

```text
SignJournalReceipt
SignJournalCheckpoint
SignControllerJob
SignADAuthorizationAssertion
SignRecoveryJob
```

The implementation constructs or validates the permitted purpose internally.

## Key separation

At minimum:

```text
Journal attestation key
    != Controller Job signing key
    != AD Authorization Broker signing key
    != Recovery Authority key
    != transport mTLS private keys
```

Private-key reuse across independent authority classes is prohibited even if Signature v1 domain separation is present.

## Signature bytes

For Ed25519, `signature` is the exact 64-byte Ed25519 signature encoded as lowercase hexadecimal in the JSON envelope unless the artifact's schema explicitly chooses base64url without padding. One encoding must be frozen per schema and cannot be autodetected.

For Journal Signature v1 uses, lowercase hexadecimal is the v1 encoding.

## Key rotation and historical verification

Rotation creates a new `signing_key_id`. Historical signatures continue to verify against the exact public key generation that created them.

Key loss, key compromise, expiration, and normal rotation are distinct facts and must not be collapsed into one state.

## Acceptance tests

At minimum test:

- correct signature/purpose/payload;
- same signature under wrong purpose;
- same payload signed by unauthorized authority class;
- altered payload bytes with same semantic JSON;
- altered `payload_sha256`;
- wrong `signing_key_id`;
- wrong algorithm declaration;
- malformed signature length/encoding;
- unknown purpose;
- old valid key generation where historical verification remains permitted; and
- revoked/compromised/untrusted key generation according to the consuming policy.
