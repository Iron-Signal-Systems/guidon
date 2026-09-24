# Active Directory Recovery Architecture

## Purpose

Active Directory is a foundational dependency for many on-premises Guidon target environments. A backup of a domain controller is not, by itself, proof that the forest can be brought back into service.

Guidon therefore treats Active Directory recovery as a first-class workload with specialized recovery artifacts, authority, isolation, orchestration, milestones, validation, and destructive testing.

## Product objective

The operational objective is intentionally simple:

> **An authorized administrator can review an exact recovery plan, start Active Directory recovery, and restore a minimum usable identity core without depending on the failed forest for recovery authority.**

For a small, pre-qualified supported environment, Guidon engineering should optimize the path from recovery authorization to a validated identity core into minutes where the underlying platform permits it. Guidon does not claim a numeric RTO until that time has been repeatedly measured on the supported topology and recovery target.

Remaining domain controllers may be rebuilt after the minimum identity core is operational.

## Governing rules

1. Active Directory recovery is not modeled as a generic Windows image restore.
2. Recovery authority must survive loss or compromise of normal Active Directory trust.
3. Destructive AD recovery actions are generated as an explicit recovery plan and authorized as an exact signed Guidon Job.
4. Recovery begins in an isolated environment until the supported health and security gates permit production reconnection.
5. Guidon distinguishes rapid operational recovery from compromise recovery.
6. A backup-success state is never projected as proof that Active Directory can be recovered.
7. Recovery readiness is established by required artifacts, current verification facts, compatible recovery procedure, available recovery authority, and actual recovery exercises.
8. Recovery time is measured from actual recovery operations; it is not inferred from backup duration.
9. Guidon follows supported Windows/AD DS recovery mechanisms for the targeted Windows Server versions rather than inventing unsupported restore shortcuts.

## Initial supported profile

The first implementation should deliberately constrain topology so the recovery procedure can be tested destructively and repeatedly.

A suitable initial profile is:

```text
one forest
one domain
two to four supported Windows Server domain controllers
AD-integrated DNS
DFSR SYSVOL
one designated primary recovery DC
one designated secondary recovery candidate
supported virtual recovery target
Guidon Recovery Authority configured and tested
```

This is an implementation starting point, not a permanent product limit.

Multi-domain forests, complex trust topologies, specialized DNS delegation, physical domain controllers, read-only domain controllers, unusual storage layouts, and other cases are added only when their recovery behavior is explicitly supported and validated.

For any supported domain controller whose complete recovery material is not supplied through a supported virtualization-platform recovery path, the Windows system-recovery architecture requires the host-local System Recovery Worker. During normal domain operation that privileged Worker runs as a gMSA and is separate from the ordinary Guidon Windows Agent. The gMSA is not relied upon as the forest-recovery authority after AD has failed; the Recovery Authority/bootstrap path remains independent.


## AD Recovery Set

Guidon groups the exact artifacts and recovery facts required for an Active Directory recovery operation into an **AD Recovery Set**.

An AD Recovery Set is not another mutable backup catalog. It is a workload-specific recovery definition that references immutable Guidon recovery data and the exact recovery procedure/profile under which those artifacts may be used.

Conceptually it binds:

```text
AD Recovery Set
|
+-- recovery_set_id
+-- forest identity
+-- domain identity
+-- recovery point identity
+-- designated primary recovery DC
+-- designated secondary recovery candidate
|
+-- full-system/BMR recovery data
+-- required System State / AD recovery material
+-- NTDS-related recovery material as supported
+-- SYSVOL recovery material
+-- AD-integrated DNS recovery material
|
+-- forest/domain functional-level observations
+-- FSMO observations
+-- Global Catalog observations
+-- site/subnet/topology observations
+-- time-hierarchy observations
+-- trust observations where applicable
|
+-- Windows/AD recovery procedure profile + version
+-- Guidon configuration/policy generation
+-- Recovery Authority requirements
|
+-- Repository verification facts
+-- Journal attestation facts
+-- last isolated recovery exercise
+-- last identity-core validation
+-- measured recovery time
```

The exact v1 schema is frozen when implementation begins. The architecture requirement is that enough immutable identity/provenance exists to determine what recovery procedure was actually applicable to the captured material.

## Designated recovery DC

Each protected domain identifies a preferred recovery DC and a secondary candidate.

The preferred recovery DC should be operationally simple and intentionally suitable for disaster recovery. Guidon records the facts it can establish about that system, including:

```text
DC identity
Windows Server version/build
domain/forest identity
DNS role
Global Catalog state
FSMO role state
site
virtual/physical recovery profile
latest compatible recovery point
latest verification result
latest isolated recovery result
```

A designation does not make a recovery point valid. The current AD Recovery Set must still satisfy all required artifact, integrity, authority, and compatibility checks.

## Recovery modes

### Rapid operational recovery

Use when the failure cause is understood and there is no declared reason to treat the forest's security state as compromised.

The goal is the shortest supported path to a validated identity core while preserving recovery correctness and auditability.

Examples may include:

```text
storage failure
hypervisor failure
failed patch/update
accidental deletion/destruction
known non-malicious logical failure
```

### Compromise recovery

Use when the forest, privileged identities, credentials, or domain-controller integrity may have been compromised.

This mode prioritizes a trustworthy recovery boundary over minimum elapsed time.

The generated plan may require additional actions such as:

```text
clean isolated recovery target
stronger recovery-point selection requirements
privileged credential reset
krbtgt reset workflow
gMSA/key remediation
trust remediation
certificate/PKI remediation
persistence review/remediation
additional validation before production reconnection
```

Guidon records which mode and policy generation were used. A rapid-recovery result is not silently represented as satisfying a compromise-recovery policy.

## Recovery authority

A complete forest failure may make normal AD authorization unavailable.

Therefore the authority to initiate forest recovery is separate from:

```text
Domain Admin
Enterprise Admin
working domain logon
gMSA availability
historic LAPS password
historic local Administrator password
```

Recovery uses the Guidon Recovery Authority defined by the identity/trust and Job/authorization contracts.

Where policy requires MFA, recovery MFA is independent of the failed forest and bound to the exact recovery authorization/Job.

Guidon must not durably store a DSRM password, entered OTP value, temporary first-boot password, or other secret merely to simplify future forest recovery.

## Recovery-plan review

Forest recovery is not a blind one-click destructive action.

Before execution, Guidon produces a plan containing the exact supported steps and targets for the selected Recovery Set and recovery mode.

A plan may include:

```text
selected forest/domain
selected recovery point
selected recovery DC
recovery mode
isolation target
system reconstruction
AD DS recovery procedure
SYSVOL authority procedure
FSMO handling
RID/metadata handling
DNS restoration
Global Catalog establishment
time-service handling
credential/security remediation required by policy
health gates
production reconnection gate
remaining-DC rebuild plan
```

The administrator authorizes the exact plan as a constrained signed Job. Material changes require a new authorization rather than silently altering an already authorized recovery operation.

## Isolation

The first recovered domain controller must be recoverable in a defined isolated recovery environment.

Isolation prevents an incompletely recovered DC from immediately interacting with unknown or damaged production peers and allows compromise-recovery procedures to establish a clean trust boundary.

Guidon must explicitly record:

```text
isolation requested
isolation established / not established
recovery network identity
production reconnection authorization
production reconnection time
```

If required isolation cannot be established, the operation fails closed for recovery modes that require it.

## Recovery sequence

The exact Windows-version-specific procedure is implementation/profile dependent, but the high-level Guidon sequence is:

```text
authorize exact recovery plan
    ->
validate AD Recovery Set
    ->
validate Repository/Journal/Recovery Authority requirements
    ->
establish isolated recovery environment
    ->
reconstruct designated recovery DC
    ->
perform supported AD DS recovery procedure
    ->
establish/validate SYSVOL
    ->
establish/validate DNS
    ->
establish required FSMO authority
    ->
establish Global Catalog where required
    ->
perform recovery-mode security remediation
    ->
validate identity-core services
    ->
disarm temporary recovery bootstrap
    ->
IDENTITY_CORE_OPERATIONAL
    ->
authorize production reconnection when policy permits
    ->
deploy/promote remaining DCs using the supported procedure
    ->
validate replication and complete forest health
    ->
FULL_FOREST_RECOVERED
```

The exact steps executed, skipped, refused, retried, or failed remain factual recovery history.

## Workload-specific milestones

Guidon introduces two AD workload milestones:

### IDENTITY_CORE_OPERATIONAL

This means the minimum supported AD identity service required by the recovery profile has been restored and validated.

For the initial profile, applicable health gates include actual checks for:

```text
AD DS service availability
SYSVOL share
NETLOGON share
required DNS zones/service
required DNS SRV registration
LDAP bind
Kerberos authentication/TGT acquisition
Global Catalog availability when required
FSMO role ownership/availability
supported dcdiag checks
temporary recovery bootstrap disarmed
```

The exact check set is versioned with the recovery profile.

A booted domain-controller VM is not sufficient.

### FULL_FOREST_RECOVERED

This is a later state.

It requires the supported remaining-DC deployment/recovery plan to complete, required replication health to be established, final forest health validation to succeed, and any recovery-mode-specific remediation to complete.

`IDENTITY_CORE_OPERATIONAL` and `FULL_FOREST_RECOVERED` do not replace Guidon's generic states such as `RESTORED` or `VALIDATED`; they describe workload-level recovery progress.

## Measured recovery readiness

Guidon should answer the administrator's practical question:

> **If my domain controllers disappear now, what can Guidon actually establish about my ability to recover Active Directory?**

A status view may expose facts such as:

```text
latest AD Recovery Set
capture time
designated recovery DC
required artifacts present
Repository state
Journal state
Recovery Authority state
procedure/profile compatibility
last isolated recovery test
last IDENTITY_CORE_OPERATIONAL validation
measured time to identity core
age of that measurement
last FULL_FOREST_RECOVERED validation
```

A stale recovery exercise remains historical fact. Guidon must not convert it into current proof if the protected configuration or required recovery artifacts have materially changed.

## Destructive testing

AD recovery is not considered implemented merely because backup and restore APIs return success.

The supported profile must be exercised by deliberately removing the production DCs in an isolated/test environment and requiring an operator who did not write the implementation to recover the forest from Guidon.

The canonical scenario begins with:

```text
known healthy forest
verified AD Recovery Set
all production DCs destroyed/unavailable
normal AD authorization unavailable
```

Success requires actual transition through the defined health gates to `IDENTITY_CORE_OPERATIONAL`, followed by the defined remaining-DC/forest procedure to `FULL_FOREST_RECOVERED`.

Repeated runs provide the measured recovery-time distribution used for operational planning.

## Catalog independence

Loss of the Guidon mutable catalog/controller must not make otherwise intact AD recovery material meaningless.

The AD Recovery Set and referenced authoritative Repository artifacts must preserve enough recovery identity/provenance that a clean supported Guidon environment can rediscover the recovery material according to the Repository Format and later Guidon-DR contracts.

Catalog convenience may accelerate discovery; it does not become the sole source of recovery meaning.

## Failure behavior

Guidon must surface and preserve specific failure conditions such as:

```text
required recovery artifact missing
artifact verification failed
Recovery Set incompatible with requested procedure
unsupported forest/domain topology
unsupported Windows build/profile
designated recovery DC unavailable/incompatible
Recovery Authority unavailable
required Journal gate unavailable
isolation not established
system reconstruction failed
SYSVOL recovery failed
DNS validation failed
LDAP validation failed
Kerberos validation failed
Global Catalog validation failed
FSMO validation failed
security remediation incomplete
temporary recovery bootstrap not disarmed
remaining-DC replication validation failed
```

Failure to establish one of these properties must not be projected as a successful identity-core recovery.

## Relationship to other Guidon architecture

This architecture inherits and does not weaken:

- Repository immutability and catalog independence;
- Record v1 factual-history rules;
- Journal attestation and gating;
- Job exact-byte/signature requirements;
- Recovery Authority separation;
- MFA exact-Job binding;
- encrypted-at-rest durable Job storage;
- transport identity and mTLS requirements;
- verification-state semantics; and
- presentation rules that prohibit converting unknown/degraded/unverified conditions into a positive state.

Active Directory recovery is an early workload-specific use of those foundations, not a parallel security model.
