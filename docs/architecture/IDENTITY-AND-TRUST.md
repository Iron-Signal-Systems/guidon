# Identity and Trust Architecture

## Principle

Guidon separates stable identities, execution identities, human identities, transport credentials, authorization assertions, and signing authorities.

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
```

Examples:

```text
Journal mTLS private key
    != Journal Ed25519 attestation key

Controller mTLS private key
    != Controller Job signing key
```

Endpoint transport private keys should normally be generated and retained on the endpoint and not exported to the Repository.

## Job signing versus authorization

A valid Controller Job signature proves that a trusted Guidon job-signing authority signed the exact Job v1 bytes. It does **not** prove that the requesting user is authorized.

For normal manually initiated privileged operations, Guidon requires the relevant authorization path separately.

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

AD unavailable, invalid response, expired context, or inability to establish the required fact is `not_determined`, not `denied` and not `authorized`.

Normal privileged operations fail closed for both `denied` and `not_determined`, while preserving the exact reason.

Internally use SIDs for authorization identity; names are descriptive observations.

The Auth Broker operates under a dedicated gMSA and records its own gMSA authentication facts.

The Broker's AD communication is encrypted/authenticated/fail-closed according to the network-security contract.

The Broker does not copy the full user group/token set unless needed for the specific authorization check.

## Break-glass Recovery Authority

Break-glass recovery does not use a stored emergency password and does not upload/store a recovery private key on normal Guidon infrastructure.

Guidon uses a dedicated offline Recovery Authority separate from:

- normal endpoint CA/transport credentials;
- Controller signing authority;
- Journal signing authority; and
- AD Authorization Broker authority.

Guidon infrastructure retains only the required public trust material.

The preferred recovery path is:

```text
offline Recovery Authority
    -> issues short-lived scoped recovery credential/certificate
    -> private key exists only in recovery environment/session
    -> proof of possession
    -> signed Recovery Job
    -> mTLS
```

The recovery credential/job is tightly scoped to the endpoint/recovery point/operation.

If desired, Guidon may record SHA-512 of the exact recovery certificate in addition to the normal thumbprint and SHA-256. The certificate digest is not confused with the certificate's signature algorithm or public-key algorithm.

Break-glass recovery records the exact certificate identity/chain/validity and participating network observations.

If AD is unavailable during disaster recovery, Guidon records AD as unavailable and records the Recovery Authority as the actual authorization source. It does not pretend AD approved the operation.

Break-glass recovery authority is recovery authority, not automatic repository-deletion authority.

## Temporary first-boot Windows recovery account

Bare-metal recovery may require a one-time signed first-boot bootstrap under LocalSystem.

The bootstrap is bound to the target, recovery point, allowed action, expiry, nonce, and signature. It creates a unique temporary local administrator with a random credential, releases that credential through the defined secure recovery path, and enforces TTL/watchdog cleanup.

Final recovery validation requires verification that:

- the temporary account is removed;
- the account is absent; and
- the bootstrap is disarmed.

Working AD trust, historic LAPS password, secure-channel repair, or immediately available gMSA are not mandatory dependencies for that recovery path.

## Signing/CA history

Certificate and signing-key rotation never rewrites historical identity.

Guidon retains the public certificates/verification keys needed to verify old records and connections for as long as those records require verification.

A known/suspected key compromise and a lost key are different conditions. Guidon records the condition actually established and does not retroactively label historical signatures beyond what can be demonstrated.
