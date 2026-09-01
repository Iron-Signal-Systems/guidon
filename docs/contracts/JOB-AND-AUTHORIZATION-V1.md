# Job and Authorization v1 Contract

## Purpose

Guidon uses constrained signed jobs for remotely initiated work. A Job v1 is a request for **one defined Guidon operation against one defined scope**.

A Job is never an arbitrary execution container.

Storage encryption and MFA requirements are defined in `JOB-STORAGE-AND-MFA-V1.md` and are part of this authorization model where applicable.

## Job v1 core rules

- `job_id` is UUIDv7.
- The exact Job v1 bytes are immutable once signed.
- Any change to the requested operation/scope creates a new job.
- Job signatures prove issuer/integrity, not human authorization.
- The job is bound to a stable Guidon target identity, not a transient certificate thumbprint.
- Jobs expire and use a cryptographically random nonce/replay-control value.
- The endpoint validates the complete job deterministically before execution.
- Every rejection creates an explicit factual Record v1.
- Job acceptance is not operation completion; execution uses a separate UUIDv7 `operation_id`.
- Any durable Guidon-controlled copy of the signed Job is encrypted at rest according to `JOB-STORAGE-AND-MFA-V1.md`.

## No arbitrary execution fields

Job v1 must not contain generic fields that recreate remote code execution, including:

```text
command
shell
powershell
script
executable
generic arguments
generic parameters bag
```

Operations are a controlled enum with operation-specific typed scopes.

Examples may include:

```text
BACKUP_WINDOWS_FILE
BACKUP_WINDOWS_DIRECTORY
BACKUP_WINDOWS_VOLUME
RESTORE_FILE
RESTORE_DIRECTORY
RESTORE_VOLUME
DELETE_RECOVERY_POINT
BACKUP_POSTGRESQL_CLUSTER
RESTORE_POSTGRESQL_CLUSTER
BACKUP_MSSQL_DATABASE
RESTORE_MSSQL_DATABASE
```

Dangerous options are explicit typed/enumerated fields, not free-form escape hatches.

## Job envelope

A Job v1 should preserve, as applicable:

```text
schema
schema_version
job_id
issued_at
expires_at
nonce
trigger
issuer
operation
target
scope
requested_by
policy/authorization requirements
```

The exact bytes are signed using the Controller Job signing authority with a detached signature under `SIGNATURE-V1.md`.

Signing happens before durable Job-storage encryption so decryption later reproduces the exact bytes that were authorized and signed.

## Trigger

Initial trigger vocabulary:

```text
manual
scheduled
api
cli
recovery
system
```

A manual job preserves the requesting user's critical identity and immutable authentication reference.

A scheduled/system job truthfully uses:

```text
requested_by.user.presence = not_present
```

and references the schedule/policy that caused it.

The gMSA that later executes the job is not the requester. Execution attribution is recorded by the component that actually performs the operation.

Scheduled/system work does not fabricate an interactive MFA event merely to satisfy a reporting field.

## Target binding

Jobs bind to stable Guidon identities such as `endpoint_id` rather than a current certificate thumbprint because certificates rotate.

The actual transport certificate used during execution is recorded separately in the connection/operation records.

## Endpoint validation

Before accepting a Job v1, the endpoint validates at least:

1. schema and supported schema version;
2. detached signature;
3. trusted Job issuer/signing key;
4. target Guidon identity;
5. supported operation;
6. issued/expiry time requirements;
7. nonce/replay state;
8. required authorization assertion where applicable;
9. assertion signature and trusted Broker/Recovery authority identity;
10. assertion binding to the exact job/job SHA-256;
11. assertion user/operation/target/scope relationship;
12. required MFA result/reference where the operation policy requires MFA;
13. assertion expiry; and
14. local prerequisites required for the defined operation.

Every rejection records the exact reason. Guidon does not collapse these into `invalid job` when a precise condition is known.

If the Job was durably queued/spooled, authenticated decryption and exact-byte recovery occur before these validations as defined in `JOB-STORAGE-AND-MFA-V1.md`.

## Separate signing authorities

The Controller Job signing key is distinct from:

```text
Journal attestation signing key
AD Authorization Broker signing key
Recovery Authority signing key
transport mTLS private keys
Job-storage encryption KEKs
TOTP/MFA secret-protection keys
Recovery Copy administrator PKI/private keys
```

Compromise of one authority does not intentionally provide the others.

## Manual privileged operation

A normal manual privileged operation requires:

```text
valid signed Job v1
+
valid current authorization assertion v1
+
required MFA proof/result when policy marks the operation as MFA-required
```

where the operation's authorization contract requires those controls.

The Job signature alone does not prove that the human requester is authorized.

A successful login or previously satisfied MFA session is not substituted for exact-Job authorization when the operation policy requires exact-Job MFA binding.

## AD Authorization Assertion v1

The assertion is a short-lived signed statement about **one user SID and one exact Job v1** based on explicit AD observations from an identified Domain Controller plus the required independent authentication/MFA observations.

It is not a reusable general administrator token.

### Identity and binding

Conceptually:

```text
schema
schema_version
assertion_id UUIDv7
issued_at
expires_at

user.account
user.domain
user.sid
user authentication record reference

job_id
job_sha256
operation
endpoint/target
scope/authorization requirement

AD source observations
MFA method/result/reference when required
result

producer/Auth Broker identity
Broker gMSA authentication reference
signing_key_id
```

The assertion binds to the exact Job SHA-256 so changing a path, endpoint, recovery point, destructive option, or other signed field invalidates the authorization.

For an MFA-required operation, the assertion is issued only after the MFA verifier has established the required TOTP result and replay checks. The OTP value itself is never placed in the assertion.

## Authorization result

The locked high-level results are:

```text
authorized
denied
not_determined
```

`not_determined` is used when Guidon cannot establish the required authorization fact, including cases such as:

- AD/DC unavailable where AD authorization is required;
- required query could not complete;
- response could not be validated;
- required current authentication context is unavailable;
- required MFA verifier is unavailable;
- required MFA result cannot be established; or
- the assertion/context expired before use.

Normal privileged operations fail closed for both `denied` and `not_determined`, while preserving the exact reason.

## AD source provenance

The Broker preserves the specific DC that answered the authorization query, including where observed:

```text
hostname
FQDN
IP address
connection_id
network observations
certificate/transport observations if applicable
```

Authorization-query DC may differ from the user's authenticating DC and the gMSA's authenticating DC. These identities are never collapsed into one inferred "the DC" value.

Use SIDs internally for authorization requirements. Names are descriptive observations.

## Broker execution identity

The AD Authorization Broker runs under a dedicated gMSA and preserves a full gMSA authentication record when available.

The Broker has narrow authority to make the defined AD authorization observation and sign the resulting assertion. It does not issue Jobs, restore data, delete recovery points, or sign Journal receipts/checkpoints.

Broker-to-AD/DC communication follows the Guidon network-security requirements: encrypted/authenticated where the selected protocol supports it, identity-bound, fail-closed for required protection, and acceptance-tested with PCAP/Wireshark.

MFA verification may be performed by the Broker or a separately defined MFA verifier, but the component boundary and signing relationship must be explicit. A component must not silently gain unrelated authority merely because it can verify a TOTP.

## MFA-required operations

`JOB-STORAGE-AND-MFA-V1.md` defines the initial TOTP profile and storage/replay rules.

Policy-defined interactive privileged operations may include:

```text
restore over original/existing target
manual recovery-point deletion
trust-anchor change
endpoint identity/certificate rebinding
Journal witness-set or signing-authority change
break-glass/recovery-bootstrap activation
temporary recovery-administrator authorization
Recovery Copy export
retention/destruction policy change
```

The list is policy-controlled by explicit operation type. Guidon does not infer MFA requirement from arbitrary UI context.

A TOTP success is bound to the exact `job_sha256` through the signed authorization assertion; it is not a reusable general privilege window.

## High-risk operations

The same Job v1 mechanism is used for high-risk operations, with stronger authorization policy where required.

Future policy may require dual authorization or stronger witness participation for selected destructive/trust actions without introducing a separate arbitrary command path.

## Break-glass recovery jobs

Break-glass is an explicit alternate authorization mode, not:

```text
if AD unavailable, skip authorization
```

Recovery uses the offline Recovery Authority path defined in the identity/trust architecture. A short-lived scoped recovery credential plus proof of possession, independent recovery MFA where required, and a signed Recovery Job authorizes only the defined recovery action.

Recovery/break-glass MFA must not require production AD or the normal Controller database to be healthy.

Break-glass recovery authority does not automatically grant repository deletion, trust-anchor administration, Journal signing-key administration, Recovery Copy deletion, or other unrelated security powers.

## Recovery Copy administration

Recovery Copy export uses the separate trust boundary defined in `../architecture/RECOVERY-COPY.md`.

The intended direction is:

```text
separate Recovery Copy trust root
+
RSA-4096 Recovery Copy administrator certificate
+
TOTP MFA
+
explicit scoped export authorization
```

Normal primary Guidon replication credentials are not recovery-export authority.

## Journal gating

Acceptance of defined privileged/destructive/recovery-writing operations is Journal-gated before the associated side effect begins.

The gating record proves that the exact authorized Job/action boundary was independently witnessed before target modification or destruction began.

Future multi-Journal availability/strong-witness policy is defined in `../architecture/MULTI-WITNESS-AND-ANCHORING.md`. The presence of multiple Journals does not change the signed Job or exact authorization relationship.
