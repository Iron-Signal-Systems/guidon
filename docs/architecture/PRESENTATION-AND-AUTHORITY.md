# Presentation Depth and Operational Authority

## Purpose

Guidon contains detailed recovery, security, identity, cryptographic, Journal, and failure-state information. That rigor must not require every operator to consume the full internal model during normal work.

This document freezes the product invariant that Guidon may present the same underlying facts at different levels of detail without changing their meaning or silently expanding a user's authority.

The governing rules are:

> **Simple presentation must not create false certainty.**

> **Presentation depth and operational authority are separate dimensions.**

## Same facts, different depth

Guidon may provide different presentation levels for different operational needs, but those views derive from the same underlying Guidon facts and state contracts.

The intended presentation levels are conceptually:

```text
LEVEL 1 — STATUS / EXECUTIVE
    What requires attention?
    Are protected systems currently recoverable according to policy?

LEVEL 2 — OPERATIONS
    What happened?
    Which systems, recovery points, copies, tests, or dependencies require action?

LEVEL 3 — TECHNICAL
    How did Guidon determine the state?
    Which Repository, Journal, witness, Recovery Copy, identity, policy, and verification facts participated?

LEVEL 4 — AUTHORITATIVE DETAIL
    Show the exact underlying records, hashes, manifests, receipts, certificates,
    key generations, configuration generations, timestamps, and failure reasons.
```

These are presentation concepts, not fixed screen designs and not security roles.

Phase 0 does not freeze navigation, widgets, dashboards, colors, page layouts, or final product terminology.

## Summary projection must preserve truth

A summary may reduce detail but must not transform a non-positive state into a positive one.

Examples:

```text
unknown
    -> remains unknown

not_observed
    -> remains not observed / unavailable to establish

not_verified
    -> remains not verified

degraded
    -> remains degraded

stale verification
    -> remains visibly stale

failed required dependency
    -> cannot be averaged into a green overall state
```

`not_applicable` is distinct from `unknown` or `not_observed`.

Guidon must not convert absence of a failure record into proof of health.

## Positive status requires defined policy satisfaction

A high-level positive state such as `HEALTHY`, `PROTECTED`, or `RECOVERABLE` may be presented only when the exact policy defining that summary is satisfied by the required underlying Guidon facts.

For example, a policy may require:

```text
latest recovery point = COMMITTED
Repository verification = current
required Journal witness state = satisfied
required Recovery Copy state = verified
required recovery test age <= policy threshold
required configuration/trust state = current
no unresolved blocking integrity conflict
```

The exact policy may vary by customer/system class, but the UI cannot invent a positive state outside the configured policy.

The administrator must be able to drill down and determine why Guidon presented the summary.

## Presentation depth is not authority

Viewing detailed security or recovery facts does not inherently grant permission to change them.

Examples:

```text
Security auditor
    -> may have deep Level 4 visibility
    -> may have no restore/delete/trust-change authority

Executive viewer
    -> may have broad organizational Level 1 visibility
    -> may have no operational authority

Recovery operator
    -> may execute defined restore operations
    -> may not manage trust roots or destroy Recovery Copy data
```

Guidon authorization therefore models capabilities/authorities independently of presentation depth.

No role name such as `administrator` is treated as automatic universal authority.

## Capability-oriented authorization direction

Future UI/RBAC implementation may compose capabilities such as:

```text
view.summary
view.protection
view.recovery_points
view.security_records
view.cryptographic_detail
view.authoritative_records

backup.manage
restore.preview
restore.execute
retention.manage
recovery_point.delete
trust.manage
certificate.manage
recovery_copy.export
audit.export
```

The exact capability schema is not frozen in Phase 0.

The invariant is that viewing and acting are independently authorized and that a broad display role does not silently become destructive authority.

## Failure and degraded-state usability

Guidon should make failure actionable rather than merely detailed.

A summary view should answer:

```text
what is wrong?
what is affected?
how long has it been true?
is recovery currently blocked or merely degraded?
what operator action is required, if any?
```

A deeper view should then expose the exact underlying failure facts.

This allows Guidon to remain simple during normal operations without hiding the details required during incident response, recovery, audit, or engineering support.

## Recovery workflow presentation

Recovery interfaces must present the operator with the material facts that change the requested action before authorization is finalized.

For privileged exact-Job-bound operations, the human-readable operation/target/scope shown for approval must correspond to the exact Job that will be signed/authorized.

A simplified recovery UI must not authorize one broad operation and later mutate the underlying Job into a different target, scope, destructive option, or recovery point without a new authorization where required by policy.

## Security/audit presentation

Security and audit views should be capable of tracing a summary state to the underlying attribution and authorization facts, including as applicable:

```text
user identity
user authentication
AD authorization assertion
MFA result/reference
gMSA/service identity
endpoint identity
transport certificate identity
Controller signing key generation
Repository/Journal identities
Journal receipt/checkpoint
record_id / record_sha256
manifest/recovery-point identity
configuration/policy generation
clock/time provenance
```

The existence of a deep view does not require normal operators to see all of these fields during routine operation.

## Cross-product direction

Iron Signal Systems products may use a similar presentation-depth concept, but Guidon remains responsible for its own state meanings and authorization model.

A shared visual language must not cause one product's state or authority semantics to be silently imported into Guidon.

## Phase 1 implications

Phase 1 does not require a polished UI.

It must preserve enough structured state and provenance so later presentation layers can accurately project:

- positive state;
- degraded state;
- unknown/not-observed state;
- verification age/currentness;
- required dependency state;
- exact failure reasons; and
- drill-down references to authoritative facts.

Implementation must not collapse these distinctions into a single boolean success field that would make truthful presentation impossible later.

## Explicit non-claims

Guidon does not claim that:

- a simplified summary replaces the authoritative records;
- executive visibility grants administrative authority;
- technical visibility grants destructive authority;
- a green state can be inferred merely because no error was displayed;
- one universal administrator role is required; or
- Phase 0 defines the final UI design.
