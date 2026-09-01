# Time and Configuration Provenance Contract

## Purpose

Guidon must be able to explain **when** something was observed/recorded and **which exact configuration/policy generation** governed the operation.

Distributed clocks and mutable configuration files are not treated as unquestionable truth.

---

# Time provenance

## Core rule

> **Guidon records timestamps as observations from identified clocks. It does not pretend timestamps from different systems establish exact global ordering unless that ordering is independently proven.**

## Record time fields

Every authoritative Record v1 preserves at least:

```text
occurred_at
recorded_at
```

where:

```text
occurred_at
    when the producing component reports/observes that the event occurred

recorded_at
    when the producing component committed the immutable Record v1
```

These may differ legitimately.

## Producer clock provenance

Where available, preserve facts such as:

```text
system
clock_type
synchronization_source_if_observed
synchronization_state_if_observed
last_successful_sync_if_observed
stratum_if_observed
offset_from_reference_if_directly_observed
```

The exact source differs by platform. Windows may expose Windows Time information; FreeBSD may expose `ntpd`/`chrony` state. Guidon records what it actually observes rather than assuming synchronization because a service is configured.

## Mandatory ISS system clock observation

Every Record v1 entering the authoritative ISS/Guidon record path must preserve a separate **ISS system clock** observation so an administrator can compare the producer's time with the receiving Guidon infrastructure's clock if necessary.

Conceptually:

```json
{
  "time": {
    "occurred_at": "...",
    "recorded_at": "...",
    "producer_clock": {
      "system": "ISS-FS-01",
      "source_if_observed": "ISS-DC01.iss.local",
      "synchronization_state": "synchronized"
    },
    "iss_system_clock": {
      "observed_at": "...",
      "system": "GUIDON-REPOSITORY-01",
      "instance_id": "019d...",
      "source_if_observed": "...",
      "synchronization_state": "synchronized"
    }
  }
}
```

At minimum `iss_system_clock` identifies:

```text
observed_at
system
instance_id
```

and includes synchronization/source facts when actually observed.

The field exists to provide an ISS-side time anchor for historical comparison. It does not replace the producer's own timestamps.

## Journal time is separate

Journal receipt/entry processing adds its own independently identified time facts, for example:

```text
Journal received_at
Journal durable_at/journaled_at
Journal system/instance
```

A typical chain may therefore preserve:

```text
SOURCE / PRODUCER CLOCK
    occurred_at
    recorded_at

ISS SYSTEM CLOCK
    observed_at when record entered Guidon authoritative path

JOURNAL CLOCK
    received_at
    durable/journaled_at
```

None replaces another.

## Do not infer clock drift from transport delay

If the producer timestamp differs from the ISS-system observation by 500 ms, Guidon may report the factual timestamp difference where useful.

It does not automatically conclude the producer clock is 500 ms wrong because the difference can include:

- record construction time;
- local queueing;
- network latency; and
- receiving-component processing time.

An actual clock-offset claim requires a direct clock comparison/measurement that establishes it.

## Historical timestamps are immutable

If a later observation establishes that a source clock was offset, Guidon does not rewrite historical `occurred_at` values.

Instead it preserves the original timestamp and adds the later clock/synchronization observation.

## Cross-system ordering

Wall-clock timestamps from different systems do not establish exact global ordering by themselves.

Provable ordering may come from:

- Journal sequence within one stream;
- explicit protocol request/response relationships within a producer;
- exact record/receipt dependencies;
- operation/job/connection/authentication correlations; or
- another defined causal relationship.

Guidon does not convert small timestamp differences into unsupported global ordering.

## Domain Controller event time

For AD-related Windows Security events, preserve the DC event's own occurrence time and event identity separately from Guidon's observation/collection time.

For example:

```text
DC event occurred_at
Event ID
EventRecordID
DC computer identity
Guidon observed_at
ISS system clock observation
Journal received_at
```

These are separate facts.

## Offline/disconnected records

If an endpoint creates a record while disconnected and Guidon receives it later, preserve both times.

Guidon does not rewrite the event as though it occurred when the Repository finally received it.

## Time synchronization changes

Meaningful time-source/synchronization observations may create records such as:

```text
TIME_SOURCE_CHANGED
CLOCK_OFFSET_OBSERVED
CLOCK_SYNCHRONIZATION_LOST
CLOCK_SYNCHRONIZATION_RESTORED
```

Only the facts actually observed are recorded. Avoid broad conclusions such as `clock_bad` without a defined testable threshold.

## UUIDv7

UUIDv7 is used for time-sortable identities/correlation but does not replace explicit timestamps or clock provenance.

---

# Configuration and policy provenance

## Core rule

> **Guidon must be able to identify the exact configuration and policy generations that governed a historical operation.**

A mutable file overwritten in place is insufficient historical provenance.

## Immutable configuration generations

Every accepted configuration generation has its own immutable identity and exact-byte SHA-256.

Conceptually:

```text
configuration_id = UUIDv7
configuration_version
configuration_sha256 = SHA-256(exact accepted configuration bytes)
created_at
activated_at
```

Changing configuration creates a new generation rather than modifying the old generation's historical identity.

The same principle applies to policies such as:

```text
retention_policy_id / version / sha256
authorization_policy_id / version / sha256
verification_policy_id / version / sha256
```

## Operations reference exact generations

Meaningful operations preserve references to the exact configuration/policy generations that governed them.

For example, a recovery point may reference:

```text
repository configuration ID/version/SHA-256
backup policy ID/version/SHA-256
verification policy ID/version/SHA-256
```

A deletion record references the exact retention-policy generation evaluated at that time.

Historical operations never use a vague `current` policy reference.

## Candidate, validated, and active configuration are separate states

Guidon distinguishes:

```text
configuration submitted
configuration validated
configuration rejected
configuration activated
```

Merely writing candidate bytes to disk does not make them active.

If generation 13 fails validation while generation 12 is active:

```text
candidate = 13
accepted = no
active = 12
```

The previous known-good active configuration remains active unless the defined transition succeeds.

## Atomic activation

The conceptual configuration activation path is:

```text
receive candidate exact bytes
    -> SHA-256
    -> validate schema
    -> validate references/constraints
    -> durably preserve immutable generation
    -> required Journal gate for security-sensitive change
    -> atomically update active-generation reference
    -> durably record activation
```

After a crash, existence of a candidate generation does not prove it was active. Guidon reconstructs the active state from durable authoritative activation facts/markers.

## Security-sensitive changes are Journal-gated

Changes affecting security/trust/destruction/recovery authority are synchronously Journal-gated before activation, including as applicable:

- trust-anchor changes;
- certificate identity/binding changes;
- Controller/Journal/Auth Broker signing-authority changes;
- Recovery Authority trust changes;
- destructive authorization requirements; and
- other configuration changes that alter a defined high-impact security boundary.

Operational tuning values may use a different activation path where appropriate, but the change still creates immutable configuration provenance records.

## Out-of-band modification

If an administrator or host process modifies active configuration bytes outside the Guidon configuration workflow, Guidon does not silently adopt the new bytes.

At startup/reload it compares the expected active generation/hash with what is actually observed and records a mismatch such as:

```text
ACTIVE_CONFIGURATION_INTEGRITY_CHECKED
expected_sha256 = AAA
observed_sha256 = BBB
result = mismatch
```

The affected operation may fail closed when the altered configuration controls a required security/recovery boundary.

Guidon never updates the catalog merely to make the unexpected bytes become the new historical authority.

## Startup verification

Every component verifies its active authoritative configuration on startup, whether the previous shutdown was clean or unclean.

This is separate from the much larger full Repository reverification required after an unclean storage/power event.

## Secrets and private keys

Immutable configuration provenance must not become a permanent secret dump.

Configuration/history records store references/identifiers such as:

```text
credential_reference
certificate_sha256
signing_key_id
secret identifier
```

rather than private keys or plaintext passwords.

Private material remains within the component/security boundary that owns it.

## Configuration versus observed behavior

Configuration states what Guidon was required/instructed to do. Records establish what actually happened.

For example:

```text
configuration: require mTLS client certificate
```

is not proof that a particular connection presented/validated one.

The connection record must preserve the actual certificate/chain/acceptance facts.

Likewise:

```text
policy: full post-failure reverification required
```

is not proof that every retained object was re-hashed. Verification-epoch records establish that work.

## Change attribution

Manual configuration/policy changes preserve the relevant user/authentication/authorization/Job provenance.

Automatic/system-generated changes truthfully record `user.presence = not_present` and identify the component/policy responsible.

An administrator who created a policy months earlier is not falsely attributed as the human actor for every later scheduled action caused by the policy.

## Policy evaluation

Policy evaluation records the exact policy generation and factual inputs used.

For retention, for example:

```text
recovery_point_id
policy_id
policy_version
policy_sha256
evaluated_at
capture time/value used
result = eligible_for_deletion / not_eligible / not_determined
```

This makes the historical evaluation reproducible/explainable without relying on today's policy contents.

## Component separation

Repository, Journal, Controller, Authorization Broker, and endpoint configuration remain separate according to their responsibility boundaries.

A single configuration package must not hand every component every secret merely for convenience.

In particular, Repository configuration does not contain the Journal attestation private key, and normal infrastructure configuration does not contain the offline Recovery Authority private key.
