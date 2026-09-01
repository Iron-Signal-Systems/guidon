# Identity and Trust Architecture

## Principle

Guidon separates stable identities, execution identities, human identities, authentication factors, transport credentials, authorization assertions, signing authorities, storage-encryption authorities, and recovery authorities.

No single credential should silently become universal authority.

## Attribution model

Guidon uses distinct first-class attribution blocks for:

```text
user
gmsa
service_identity
endpoint_identity
transport_identity
producer
network_observation
```

MFA observations are authentication/authorization facts associated with an identity and one authorization event; a TOTP factor is not represented as a user.

A gMSA is not represented as a user.

Scheduled operations truthfully use:

```text
user.presence = not_present
```

and record the gMSA/service identity that actually executed the work.

For AD-connected user and gMSA identities, preserve available factual fields such as:

- account;
- domain;
- SID;
- UPN/display name when observed/relevant;
- provenance for sourced fields; and
- authentication context.

## AD authentication records

Dedicated authentication records are preferred, for example:

```text
AD_USER_AUTHENTICATED
AD_GMSA_AUTHENTICATED
```

Each authentication event has a UUIDv7 `authentication_id`.

Where observed, preserve:

- authenticating DC hostname/FQDN/IP;
- protocol/package such as Kerberos or NTLM;
- Windows event channel;
- Event ID;
- EventRecordID;
- Logon ID;
- Logon GUID;
- event occurrence time;
- Guidon observation time; and
- factual network observations.

User authentication, gMSA authentication, and a later authorization query may legitimately involve different DCs. Each is preserved separately.

Guidon never claims Windows Security-event correlation exists when the needed event was not observed.

## Endpoint identity

Each protected endpoint has a stable Guidon endpoint UUID.

The endpoint UUID is separate from:

- hostname;
- IP address;
- MAC address;
- machine SID;
- current certificate; and
- current gMSA.

Those may be observed attributes/credentials and may change over time.

## Endpoint certificate identity

For server/workstation transport identity, Guidon correlates:

```text
stable endpoint UUID
    + locally observed Windows machine certificate
    + validated certificate chain
    + certificate actually presented through mTLS
```

Certificate thumbprint/history is first-class because certificates rotate.

For the exact certificate used in a relevant operation/connection, preserve as available:

```text
store location/name
Windows-reported thumbprint
thumbprint algorithm
Guidon-calculated certificate_sha256 over exact DER bytes
serial number
subject
issuer
subject/authority key identifiers where useful
not_before
not_after
EKU/key usage
public-key algorithm
signature algorithm
chain certificate SHA-256 values
validation/revocation facts actually performed
```

A Windows-reported SHA-1 thumbprint remains identified as SHA-1. Guidon must not relabel it as SHA-256. The independent `certificate_sha256` is calculated from the exact DER certificate bytes.

The Repository independently records the certificate presented during mTLS. If exact certificate SHA-256 values match between the endpoint's local observation and the presented certificate, Guidon may record that deterministic comparison as `match: true`. It does not turn that fact into a broad statement that the machine is therefore trustworthy.

## Certificate lifecycle

Stable Guidon identity and renewable certificate credentials remain separate.

Normal certificate renewal may temporarily authorize an overlap of old/new certificates for the same endpoint identity. Every connection still records which certificate actually participated.

If Guidon only observes certificate A previously and certificate B now, it records those observations. It does not claim a successful rotation process unless it actually observed/orchestrated that process.

Expired, revoked, untrusted, or identity-mismatched certificates are rejected according to the defined transport policy, and the exact rejection reason is recorded.

Trust-anchor changes and endpoint-certificate authorization/binding changes are security-sensitive, immutable, Journal-gated operations.

A certificate chaining to an approved CA is not sufficient by itself. It must also be authorized to represent the Guidon identity claimed on that connection.

The exact X.509 encoding used to bind an endpoint UUID may be selected during implementation, but the cryptographic identity-binding requirement is fixed.

## Separate certificate/key purposes

Private keys are not reused across independent authority classes.

At minimum, distinguish:

```text
endpoint transport identity
infrastructure transport identity
Controller Job signing
Journal attestation signing
AD Authorization Broker signing
Recovery Authority
Recovery Copy administration PKI
Job-storage encryption KEKs
MFA/TOTP secret-protection keys
storage-encryption keys
```

Examples:

```text
Journal mTLS private key
    != Journal Ed25519 attestation key

Controller mTLS private key
    != Controller Job signing key

Controller Job signing key
    != Controller Job-storage KEK

normal Guidon PKI root
    != Recovery Copy administration root
```

Endpoint transport private keys should normally be generated and retained on the endpoint and not exported to the Repository.

## Job signing versus authorization

A valid Controller Job signature proves that a trusted Guidon job-signing authority signed the exact Job v1 bytes. It does **not** prove that the requesting user is authorized.

For normal manually initiated privileged operations, Guidon requires the relevant authorization path separately.

Any durable Job copy is encrypted according to `../contracts/JOB-STORAGE-AND-MFA-V1.md`; possession of a Job-storage decryption key does not become Job signing authority.

## MFA/TOTP factor model

Guidon MFA v1 uses TOTP for policy-defined interactive privileged operations as defined in `../contracts/JOB-STORAGE-AND-MFA-V1.md`.

The initial v1 timestep is 30 seconds.

A successful TOTP verification is a factual statement that the configured verifier accepted the enrolled factor at a specific time/context. It is not proof of broad human intent and is not a reusable administrator identity.

The exact OTP value is never retained in Record v1, Journal, Job bodies, logs, support bundles, or durable replay state.

The verifier preserves only non-secret facts needed to explain the authorization, such as:

```text
mfa_method = totp
mfa_result
mfa_identity/token reference
verified_at
verifier identity/instance
verifier clock provenance
accepted time-step/replay reference
job_id
job_sha256
```

The TOTP shared secret is encrypted at rest under an MFA-verifier-specific key boundary and is not stored as ordinary configuration.

## Exact-Job MFA binding

TOTP itself does not contain the Job hash. Guidon creates the binding after successful factor verification by issuing/signing the authorization assertion for the exact:

```text
job_id
job_sha256
operation
target/scope
MFA verification reference/result
```

Changing the Job bytes invalidates the authorization relationship.

For high-risk operations that require exact-Job MFA, Guidon does not substitute a generic `MFA was completed recently` session window for that binding.

## AD Authorization Assertion v1

The AD Authorization Broker creates a short-lived signed assertion about **one user SID and one exact Guidon job**.

It is not a general statement that the user is globally trusted or an administrator.

The assertion is bound to the exact job, preferably including its SHA-256, so changing endpoint, operation, recovery point, path, or destructive option invalidates the authorization relationship.

Conceptually preserve:

```text
schema/schema_version
assertion_id UUIDv7
issued_at/expires_at
user account/domain/SID
user authentication record reference
job_id
job_sha256
operation
endpoint/target scope
authorization requirement/policy
required SID/group fact
AD source/DC observations
MFA method/result/reference when required
result
producer/Auth Broker identity
Broker gMSA authentication record reference
signing_key_id
```

Authorization results are explicitly separated:

```text
authorized
denied
not_determined
```

AD unavailable, invalid response, expired context, inability to establish the required fact, or required MFA verifier unavailability is `not_determined`, not silently `authorized`.

Normal privileged operations fail closed for both `denied` and `not_determined`, while preserving the exact reason.

Internally use SIDs for authorization identity; names are descriptive observations.

The Auth Broker operates under a dedicated gMSA and records its own gMSA authentication facts.

The Broker's AD communication is encrypted/authenticated/fail-closed according to the network-security contract.

The Broker does not copy the full user group/token set unless needed for the specific authorization check.

## Normal versus recovery MFA

Guidon deliberately separates normal-domain authorization from disaster-recovery authorization.

Normal path:

```text
human identity/authentication
    + AD authorization where required
    + TOTP MFA where policy requires
    -> exact-Job-bound authorization assertion
```

Recovery path:

```text
offline/separate Recovery Authority
    + recovery credential/proof of possession
    + independent recovery TOTP where policy requires
    -> exact Recovery Job authorization
```

Production AD or the normal Controller database being unavailable does not cause recovery MFA to be skipped. Recovery MFA material is maintained under its own recovery-capable trust boundary.

## Break-glass Recovery Authority

Break-glass recovery does not use a stored emergency password and does not upload/store a recovery private key on normal Guidon infrastructure.

Guidon uses a dedicated offline Recovery Authority separate from:

- normal endpoint CA/transport credentials;
- Controller signing authority;
- Journal signing authority;
- AD Authorization Broker authority; and
- Recovery Copy administration authority.

Guidon infrastructure retains only the required public trust material.

The preferred recovery path is:

```text
offline Recovery Authority
    -> issues short-lived scoped recovery credential/certificate
    -> private key exists only in recovery environment/session
    -> proof of possession
    -> required independent recovery MFA
    -> signed Recovery Job
    -> mTLS
```

The recovery credential/job is tightly scoped to the endpoint/recovery point/operation.

If desired, Guidon may record SHA-512 of the exact recovery certificate in addition to the normal thumbprint and SHA-256. The certificate digest is not confused with the certificate's signature algorithm or public-key algorithm.

Break-glass recovery records the exact certificate identity/chain/validity and participating network observations.

If AD is unavailable during disaster recovery, Guidon records AD as unavailable and records the Recovery Authority as the actual authorization source. It does not pretend AD approved the operation.

Break-glass recovery authority is recovery authority, not automatic repository-deletion or Recovery Copy export/deletion authority.

## Recovery Copy administration trust root

The Recovery Copy appliance uses a separate administrative/export PKI under `RECOVERY-COPY.md`.

The intended hierarchy is:

```text
Offline Recovery Copy Root CA
    -> Recovery Copy Administration Intermediate CA
        -> RSA-4096 Recovery Copy administrator certificate
```

The root private key should normally remain offline.

This trust root is deliberately independent of the normal Guidon endpoint/Repository/Journal transport hierarchy.

Authentication to Recovery Copy export administration requires the valid separate administrator identity plus required MFA and explicit scoped authorization. A normal Guidon replication certificate is not export authority.

Compromise of the normal Guidon CA should therefore not, by design, mint a certificate accepted as Recovery Copy recovery-administration authority.

## Temporary first-boot Windows recovery account

Bare-metal recovery may require a one-time signed first-boot bootstrap under LocalSystem.

The bootstrap is bound to the target, recovery point, allowed action, expiry, nonce, signature, and required recovery authorization/MFA facts. It creates a unique temporary local administrator with a random credential, releases that credential through the defined secure recovery path, and enforces TTL/watchdog cleanup.

Durable Recovery Jobs and temporary recovery credentials are protected according to their encryption contracts; the temporary credential is not treated as safe merely because the enclosing Job is encrypted.

Final recovery validation requires verification that:

- the temporary account is removed;
- the account is absent; and
- the bootstrap is disarmed.

Working AD trust, historic LAPS password, secure-channel repair, or immediately available gMSA are not mandatory dependencies for that recovery path.

## Signing/CA history

Certificate and signing-key rotation never rewrites historical identity.

Guidon retains the public certificates/verification keys needed to verify old records and connections for as long as those records require verification.

A known/suspected key compromise and a lost key are different conditions. Guidon records the condition actually established and does not retroactively label historical signatures beyond what can be demonstrated.

Recovery Copy administration CA history and independent Journal/external-witness verification-key history follow the same principle within their own trust domains.
