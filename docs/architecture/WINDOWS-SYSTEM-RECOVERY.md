# Windows System Recovery Architecture

## Purpose

Guidon must be able to protect and reconstruct supported Windows systems without turning every endpoint component into a highly privileged remote-administration service.

This architecture defines:

- when Guidon uses a virtualization-platform recovery path;
- when a host-local privileged recovery component is mandatory;
- the separation between the ordinary Windows Agent and the System Recovery Worker;
- the mandatory gMSA execution model for the Worker on domain-joined Windows;
- the boundary between normal backup execution identity and catastrophic Recovery Authority; and
- the privileged-operation limits that keep the Worker from becoming a generic remote shell.

## Governing rule

> **Guidon uses the lowest-authority supported capture path that can establish the requested recovery properties. When a supported virtualization-platform integration cannot provide the required full-system protection, a dedicated host-local System Recovery Worker is required. On a domain-joined Windows system, that Worker runs as a gMSA and not as a named or password-bearing user/service account.**

The capture mechanism may differ by platform. The recovery semantics may not.

## Capture-path selection

Guidon distinguishes two full-system protection paths.

### Supported virtualization-platform path

Where a supported hypervisor integration can provide the complete required system-recovery material and consistency/provenance for the requested Guidon recovery profile, Guidon may perform the full-system capture through that platform.

Examples may include supported VMware vSphere, Microsoft Hyper-V, or Proxmox VE integrations once their applicable Guidon phases are implemented and validated.

Use of a platform path does not waive guest/application consistency requirements. Guidon reports only the VSS, application-consistency, change-tracking, snapshot, or guest-state facts actually established.

A host-local privileged Worker is not installed or activated merely for symmetry when the selected supported platform path does not require it.

### Host-local System Recovery Worker path

The Worker is required when the requested full-system recovery level cannot be established through a supported virtualization-platform integration.

This includes, as applicable:

```text
physical Windows servers
standalone/non-platform-managed Windows systems
virtual Windows systems on unsupported or unavailable hypervisor integrations
supported virtual systems where the platform path cannot establish a required recovery property
```

For the initial domain-connected Windows recovery model, full-system protection of such systems requires the host-local Worker.

## Component separation

Guidon uses separate Windows components with different authorities.

```text
Guidon Windows Agent
    ordinary endpoint control / inventory / transport
    least privilege
    endpoint identity / mTLS participation
    no implied full-system backup/restore authority

Guidon System Recovery Worker
    privileged host-local recovery helper
    full-system / VSS / System State / BMR operations
    narrow typed request surface
    gMSA execution identity on domain-joined Windows
```

Installing the Worker does not elevate the Agent.

Compromise of the ordinary Agent must not automatically provide direct invocation of arbitrary Worker capabilities outside the authenticated/authorized local protocol.

## Worker execution identity

On a domain-joined Windows system, the System Recovery Worker **must run as a Group Managed Service Account (gMSA)** during normal operation.

The following are not permitted as the normal Worker execution identity:

```text
named administrator account
named technician account
shared user account
conventional domain service account with a static/admin-managed password
stored Domain Admin credential
stored Enterprise Admin credential
stored local Administrator credential
```

The gMSA is a machine/service execution identity. It is never represented as a human user.

Scheduled Worker operations truthfully record:

```text
user.presence = not_present
gmsa = exact Worker gMSA
service_identity = exact Worker service instance
endpoint_identity = exact protected endpoint
```

## gMSA scope

The domain computers permitted to use a Worker gMSA are explicitly bounded.

Guidon deployment should prefer the smallest operationally reasonable authorized-host set.

A single organization-wide recovery gMSA usable on every Windows server is not the default merely because it simplifies deployment.

Whether a deployment uses one gMSA per host or a narrowly scoped group of equivalent hosts is a deployment/profile decision, but the authorized host set must be deliberate, reviewable, and recorded as configuration provenance.

## Worker privilege boundary

The Worker receives the privileges needed for its defined recovery responsibilities.

Depending on the supported Windows recovery profile, those responsibilities may include:

```text
VSS coordination
System State capture
protected system-file access
boot/system-volume capture
security descriptor / ownership preservation
full-system/BMR capture
supported restore writes
boot/recovery metadata restoration
post-restore recovery operations
```

Guidon prefers explicit Windows service rights/privileges and narrowly scoped ACL/API authority where they can establish the required property.

Membership in a broad administrative group is not added merely as a shortcut. If Windows platform testing demonstrates that a supported recovery function requires broader authority, the supported recovery profile documents that requirement and its security effect.

The implementation must preserve the native Windows failure/result information necessary to determine which required privilege or operation failed.

## Worker request surface

The System Recovery Worker is a privileged helper, not a remote administration product.

The Worker exposes typed operations for defined recovery functions, conceptually such as:

```text
prepare snapshot
capture System State
capture protected extents/metadata
finalize capture
prepare supported restore
write supported restore extents/metadata
finalize restore
report exact result
```

The exact implementation contract is frozen when Phase 3 code begins.

The Worker must not provide a general interface equivalent to:

```text
RunCommand(...)
RunPowerShell(...)
ExecuteArbitraryFile(...)
OpenInteractiveShell(...)
GenericRemoteAdmin(...)
```

Guidon must not smuggle arbitrary execution through a loosely typed "command", "script", "arguments", or equivalent escape hatch.

## Agent-to-Worker communication

Communication between the Windows Agent/control path and the Worker is authenticated, local, narrowly scoped, replay-aware, and bound to the applicable Guidon operation/job.

A privileged Worker request carries or references enough identity/provenance to bind at least:

```text
operation_id
job_id where applicable
job_sha256 / authorization reference where applicable
endpoint_id
requested operation type
requested scope
configuration/policy generation
expiry/lifetime
replay/nonce context where required
```

The Worker independently validates the parts of the request needed to establish that it is authorized to perform that exact privileged operation.

A compromised unprivileged Agent process is not treated as authorization merely because it can reach local IPC.

## Transport identity is separate

The Worker gMSA is not Guidon's endpoint transport identity.

Guidon continues to distinguish:

```text
stable endpoint UUID
endpoint certificate / mTLS identity
Windows Agent service identity
System Recovery Worker gMSA
human authorization
Controller Job signing authority
Recovery Authority
Journal attestation authority
```

No one identity silently substitutes for another.

## Normal operation versus catastrophic recovery

The Worker gMSA is the privileged **normal-operation execution identity** for host-local full-system capture and supported restore preparation.

It is not the catastrophic recovery root of trust.

A disaster may include:

```text
Active Directory unavailable
domain controllers destroyed
machine trust unavailable
gMSA retrieval/authentication unavailable
protected Windows host destroyed
```

Therefore Guidon cannot require a live gMSA to bootstrap recovery of a system whose normal domain environment no longer exists.

The disaster path is:

```text
Guidon Recovery Authority
    -> exact authorized Recovery Job
    -> recovery environment / recovery media
    -> controlled recovery bootstrap
    -> reconstruct Windows
    -> restore required system/workload state
    -> restore/re-establish AD/trust as applicable
    -> disarm/remove temporary recovery bootstrap
    -> return normal System Recovery Worker to gMSA execution identity
```

The temporary first-boot recovery account/bootstrap defined by the identity architecture remains exceptional and recovery-scoped. It does not become a replacement for the normal Worker gMSA.

## Physical systems

For a supported physical domain-joined Windows server, Guidon cannot delegate full-system capture to a hypervisor.

The normal protection path therefore includes:

```text
Guidon Windows Agent
    +
Guidon System Recovery Worker (gMSA)
    +
supported Windows-native capture mechanisms
    ->
Guidon Repository
```

Physical recovery still requires a defined hardware/storage/boot compatibility profile. Presence of a Worker and a successful backup does not prove bare-metal portability to arbitrary hardware.

## Non-domain Windows systems

The initial Worker identity rule is specifically designed for domain-joined Windows where gMSA is available.

A non-domain/workgroup Windows system cannot satisfy a mandatory gMSA requirement. Guidon must therefore not silently substitute a named password-bearing local account and claim the same security model.

Support for non-domain full-system protection requires a separately defined execution-identity profile before that configuration is considered supported.

Until such a profile exists, a non-domain system that requires the host-local Worker is outside the supported full-system recovery profile.

## Failure behavior

Guidon surfaces specific states when it cannot establish the required Worker boundary.

Examples include:

```text
required System Recovery Worker not installed
Worker service not running
Worker gMSA not authorized for host
Worker gMSA authentication unavailable
required Windows privilege not held
Agent-to-Worker authentication failed
request expired
request replay detected
operation outside Worker contract
unsupported capture path
virtualization path unavailable
VSS/System State/full-system operation failed
recovery bootstrap not disarmed
```

Guidon does not fall back to a named user account, stored administrator password, generic shell, or weaker capture method merely to make the job appear successful.

## Testing requirements

Phase 3 testing must demonstrate both the capability and the boundary.

At minimum:

```text
supported platform-managed VM
    -> platform capture selected where sufficient
    -> no unnecessary Worker privilege used

supported physical/non-platform-managed domain-joined Windows
    -> Worker required
    -> Worker runs as approved gMSA
    -> Agent remains least privilege
    -> Agent cannot directly perform Worker-only native operations
    -> wrong/unauthorized gMSA fails closed
    -> named/password-bearing service identity is rejected
    -> arbitrary command/PowerShell execution is unavailable
    -> required full-system material is captured

catastrophic recovery
    -> AD/gMSA unavailable
    -> Recovery Authority/bootstrap path still initiates supported recovery
    -> normal Worker returns to gMSA identity only after normal trust is restored
```

## Relationship to Active Directory recovery

Active Directory recovery inherits this architecture.

A physical domain controller, or a domain controller whose complete recovery material is not supplied by a supported virtualization-platform path, requires the System Recovery Worker during normal protection.

That Worker captures under its gMSA execution identity.

After forest failure, however, Guidon does not require that failed forest to authenticate the same gMSA before forest recovery can begin. Active Directory recovery instead uses the independent Recovery Authority and recovery bootstrap defined by the AD recovery architecture.

## Relationship to other Guidon architecture

This document does not weaken:

- least-privilege endpoint design;
- constrained signed Job execution;
- exact-Job authorization/MFA;
- Repository immutability and verification;
- Journal attestation;
- endpoint certificate identity;
- Recovery Authority separation;
- temporary first-boot account cleanup requirements; or
- the prohibition on turning Guidon into a generic remote-administration framework.
