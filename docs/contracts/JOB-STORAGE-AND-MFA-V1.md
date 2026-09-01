# Job Storage and MFA v1 Contract

## Purpose

This contract defines two security properties for Guidon-controlled Jobs:

1. a finalized signed Job must not be durably stored in plaintext; and
2. policy-defined interactive privileged operations require step-up multi-factor authorization bound to the exact Job being authorized.

These properties complement `JOB-AND-AUTHORIZATION-V1.md`. Job signing, human authorization, MFA, storage encryption, and transport encryption remain separate controls.

## Governing rules

> **A Guidon Job is signed before it is encrypted for durable storage.**

> **No Guidon component may intentionally persist a complete plaintext Job body to durable storage.**

> **An MFA success authorizes only the exact Job/policy relationship for which the authorization assertion was created; it is not a reusable privileged session token.**

> **The OTP value itself is never stored in the Job, Record v1, Journal, logs, crash artifacts, support bundles, or durable replay state.**

## Job lifecycle

The normal lifecycle is:

```text
construct typed Job v1
    -> finalize exact Job bytes
    -> SHA-256 exact Job bytes
    -> create/verify required authorization relationship
    -> sign exact Job under Signature v1
    -> encrypt signed Job artifact for durable storage
    -> durably queue/spool
    -> deliver over authenticated encrypted transport
    -> if durable receiving spool is required, persist only encrypted form
    -> decrypt into controlled memory
    -> recover exact signed Job bytes
    -> verify signature
    -> verify authorization/MFA/target/scope/expiry/replay
    -> Journal-gate where required
    -> execute defined operation
    -> record factual result
    -> destroy/release plaintext working buffers as soon as practical
```

Encryption never changes the exact signed Job bytes. After decryption the implementation must recover the exact byte sequence that was originally signed.

## What counts as durable storage

The plaintext prohibition applies to any Guidon-controlled durable persistence path, including as applicable:

```text
Controller job queues
Controller databases
retry/reconciliation spools
Repository pending-operation state
endpoint receive/retry spools
Windows service state
recovery-environment state
Recovery Copy export/job state
snapshots of Guidon-controlled application data
application-generated temporary files
support bundles
normal application logs
```

If an implementation uses a database, WAL, transaction log, temporary database file, or similar mechanism, plaintext Job bodies must not leak through that mechanism.

## Plaintext memory boundary

Plaintext Job bytes may exist only for the minimum processing period required to validate/execute the Job.

Implementations must prevent avoidable persistence through:

- plaintext temporary files;
- application crash dumps containing Job bodies;
- debug logging of Job bodies;
- support bundles containing Job bodies; and
- application-controlled spool files.

Where the platform can page plaintext process memory to disk, the implementation must either use an appropriate non-pageable/locked-memory technique for sensitive Job buffers or rely on a documented encrypted pagefile/swap boundary whose limitations are tested and stated.

Guidon does not claim protection from a fully compromised running OS/root/SYSTEM authority that can inspect process memory or loaded decryption keys.

## Encrypted Job Envelope v1

The durable representation is an encrypted envelope around the already-signed Job artifact.

Conceptually:

```text
schema = guidon.encrypted-job
schema_version = 1
envelope_id = UUIDv7
job_id = UUIDv7
created_at
encryption_algorithm = aes-256-gcm
wrapping_algorithm
kek_id
nonce
wrapped_dek
ciphertext
authentication_tag
```

Outer metadata is deliberately minimal. Operation, target, scope, requested user, recovery-point identity, and other sensitive Job semantics remain inside ciphertext unless another contract explicitly requires a particular field outside the envelope.

The envelope encoding follows `EXACT-BYTE-ENCODING-V1.md`.

## Payload encryption

Job payload encryption uses:

```text
AES-256-GCM
```

with a cryptographically random 256-bit data-encryption key (`DEK`) per encrypted Job artifact.

A nonce must never repeat for the same AES-GCM key. Nonces are generated using a cryptographically secure random source unless a later implementation contract defines an equally safe deterministic construction.

The authenticated data binds at least the fixed envelope schema/version and the non-secret locator metadata required by the implementation so those fields cannot be silently substituted independently of the ciphertext.

## DEK protection

The per-Job DEK is not stored in plaintext beside the ciphertext.

The DEK is wrapped/protected by a component-specific key-encryption key (`KEK`) identified by `kek_id`.

The exact KEK storage/wrapping mechanism is platform-specific but must be frozen and acceptance-tested before the component ships. It may use an operating-system secure-key facility, TPM/HSM-backed facility where justified, or a locally protected symmetric key hierarchy.

The following key separation is mandatory:

```text
Controller Job-storage KEK
    != endpoint Job-storage KEK
    != Recovery Copy Job-storage KEK
    != recovery/bootstrap Job-storage KEK
    != Repository/ZFS storage-encryption key
    != Controller Job signing key
    != Journal signing key
    != transport private keys
```

Compromise of one Job-storage KEK must not intentionally decrypt durable Job queues belonging to another independent component/security domain.

## Key rotation

A new KEK generation receives a new `kek_id`.

Rotation may re-wrap retained DEKs without changing the exact signed Job payload or its `job_sha256`.

Key loss, key compromise, normal rotation, and key retirement are distinct factual conditions.

Guidon must not claim a queued encrypted Job is recoverable if the required KEK generation is unavailable.

## Decryption validation order

For a durable encrypted Job artifact, the consumer performs at least:

1. strict encrypted-envelope parsing;
2. supported schema/version check;
3. allowed encryption/wrapping algorithm check;
4. trusted local KEK lookup;
5. DEK unwrap/protection validation;
6. AES-GCM authenticated decryption;
7. exact signed-Job byte recovery;
8. exact Job SHA-256 calculation;
9. detached Job signature verification under `SIGNATURE-V1.md`;
10. Job v1 semantic validation;
11. authorization/MFA assertion validation where required; and
12. replay/expiry/local-prerequisite validation.

Failure at any stage fails closed for execution.

## MFA v1 purpose

MFA v1 is used for policy-defined **interactive privileged human operations**. It is not required merely because Guidon performs automated scheduled work.

Examples expected to require interactive MFA include, according to policy:

```text
restore over an existing/original target
delete recovery point
physical object deletion/GC authorization where manually initiated
trust-anchor change
endpoint identity/certificate rebinding
Journal witness-set change
signing-authority transition
break-glass activation
Recovery Copy export
recovery bootstrap activation
temporary recovery-administrator authorization
retention/destruction policy change
```

Scheduled backups and other explicitly policy-authorized unattended operations use truthful `user.presence = not_present` attribution and do not manufacture an interactive MFA event.

## MFA factor independence

Guidon uses the term **MFA** only when the authorization path establishes factors from independent authentication categories according to the deployed policy.

A client certificate/private key and a TOTP token can both be possession factors. Their mere combination is therefore not automatically described as multi-factor authentication.

Guidon's initial normal interactive model is expected to use a human authentication factor such as a password/PIN/passphrase or equivalent user-authentication proof plus the independent TOTP factor.

For Recovery Copy administration, the RSA-4096 certificate establishes the separate recovery administrative trust domain. The administrative login/unlock path must also include a knowledge/biometric or otherwise independently classified factor as defined by the implementation policy, with TOTP used as the OTP/step-up factor for protected export operations.

If the RSA private key is hardware-protected and requires a PIN, the implementation must state exactly which factors it counts and why rather than simply labeling `certificate + TOTP` as MFA.

Guidon records the factor methods/results it actually established and does not turn possession of multiple credentials from the same factor category into a stronger claim than warranted.

## TOTP v1

Initial Guidon interactive OTP uses RFC 6238-style time-based one-time passwords (`TOTP`).

The v1 time step is:

```text
30 seconds
```

A 60-second deployment profile is not part of the initial v1 contract. If later operational testing justifies it, it requires an explicit policy/profile change rather than silent per-component behavior.

The accepted skew window must be bounded and explicitly configured. A verifier may accept adjacent time steps to tolerate demonstrated clock skew, but it records which verifier clock/time-step decision was used.

The exact TOTP digit count, HMAC algorithm, secret length, provisioning format, and token lifecycle profile must be frozen before the first MFA implementation is released; implementations may not autodetect or silently accept multiple profiles for one enrolled token.

## TOTP replay protection

A TOTP value/time-step accepted for one privileged Job authorization must not be reusable to authorize another Job.

Durable replay state stores only non-secret replay facts such as:

```text
mfa_identity_id
token_id or non-secret token reference
accepted_time_step
authorization_id
job_id
job_sha256
accepted_at
```

It never stores the OTP value.

If the configured replay rule treats a time step as consumed, a second authorization attempt using the same token/time step fails closed and the exact reason is recorded.

## Exact-Job MFA binding

TOTP verification itself proves possession of the enrolled OTP factor at the verification time. It does not inherently contain the Job hash.

Guidon binds MFA to the Job by creating the authorization assertion **after** successful MFA verification and including the exact:

```text
job_id
job_sha256
operation
target/scope relationship
MFA method/result/time/reference
```

The signed authorization assertion therefore authorizes one exact Job. Changing the Job bytes invalidates the relationship.

Guidon should not use a broad time-window such as `MFA satisfied for the next 30 minutes` as a substitute for exact-Job authorization for the defined high-risk operations.

## MFA records

Record v1 may preserve:

```text
authentication_id / authorization_id
user identity/SID where applicable
mfa_method = totp
mfa_result = verified / rejected / not_determined
mfa_token_reference (non-secret)
verified_at
verifier identity
verifier clock provenance
accepted_time_step or replay-state reference where needed
job_id
job_sha256
reason/result
```

Record v1 must not preserve:

```text
OTP value
TOTP shared secret
unencrypted recovery credential
private key
```

## TOTP secret protection

TOTP shared secrets are security-sensitive authentication material and must be encrypted at rest under an MFA-verifier-specific key boundary.

TOTP secret protection keys are separate from Job-storage KEKs and normal Guidon signing/transport keys.

Recovery/break-glass MFA material must not depend on production Active Directory or the normal Controller database being available.

## Normal versus recovery MFA domains

Guidon distinguishes:

```text
Normal interactive authorization
    user identity/authentication
    + independent human authentication factor
    + normal Guidon/AD authorization path
    + TOTP factor where policy requires

Recovery / break-glass authorization
    Recovery Authority identity
    + independently classified human authentication factor
    + independent recovery MFA/TOTP verification
    + exact Recovery Job binding
```

Production AD unavailability is not a reason to skip MFA on a recovery operation whose recovery policy requires it.

## Recovery Copy administration

Recovery Copy administrative/export authorization uses its separate recovery trust domain.

The intended v1 direction is:

```text
Recovery Copy administrator certificate
    RSA-4096
    chains to separate Recovery Copy trust root
+
independent administrator login/unlock factor
+
TOTP step-up factor
+
explicit scoped export Job/request
```

Normal Guidon endpoint/Repository/Journal credentials do not satisfy this administrative trust boundary.

Possession of the RSA-4096 certificate authenticates the separate recovery-domain credential relationship; export authorization remains separately evaluated, MFA-protected according to the factor policy, and recorded.

## Fail-closed behavior

For an operation requiring MFA, all of the following block execution:

```text
wrong OTP
expired/out-of-window OTP
replayed/consumed TOTP time step
unknown/disabled token
missing required independent authentication factor
MFA verifier unavailable
required verifier clock state not trustworthy enough for configured policy
missing MFA assertion
MFA assertion bound to a different Job
expired authorization assertion
```

Guidon records `denied` or `not_determined` according to what it actually established and sets the operation to `not_performed`.

There is no automatic `MFA unavailable -> continue anyway` fallback.

## Acceptance tests

At minimum test:

- encrypted Job at rest contains no plaintext operation/target/scope body;
- offline copy of Job storage cannot decrypt with the wrong component KEK;
- altered ciphertext/tag/nonce/envelope authenticated fields fail closed;
- successful decrypt reproduces byte-for-byte the signed Job that was originally stored;
- plaintext Job is not written to application temp files/logs/support bundles;
- normal automated job records `user.presence = not_present` and does not fabricate MFA;
- valid TOTP plus the required independent factor authorizes the intended exact Job;
- same TOTP/time step replayed for another Job is rejected;
- valid TOTP followed by any Job-byte modification invalidates authorization binding;
- previous/next time-step behavior matches the configured skew policy;
- OTP value and TOTP secret never appear in Record v1/Journal output;
- MFA verifier outage fails closed for required operations;
- Recovery Copy does not count certificate + TOTP as different factor categories unless the implemented authentication mechanism actually establishes independent categories; and
- recovery MFA remains usable when production AD is unavailable.
