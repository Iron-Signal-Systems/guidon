# Initial Threat Model

## Purpose

This threat model states what Guidon intends to prevent, detect, fail closed against, record, recover from, or explicitly does not claim to survive.

The governing rule is:

> **Guidon never turns a security assumption into a security claim. A boundary is trusted only to the degree the design can actually enforce or verify it.**

## Response vocabulary

Each significant threat should be classified using one or more explicit responses:

```text
PREVENT
    architecture is intended to stop the action

DETECT
    the action/discrepancy may occur but Guidon is intended to expose it

FAIL CLOSED
    required authority/verification unavailable means the protected action is not performed

RECORD
    Guidon preserves the factual event even when it cannot prevent it

RECOVER
    a defined reconciliation/recovery path exists

OUTSIDE CLAIM
    Guidon explicitly does not claim protection from the condition
```

Avoid vague labels such as `secured` or `mitigated` without stating what Guidon actually does.

## Trust boundaries

Initial Phase 1 boundaries include:

```text
Protected Windows Endpoint
        |
        | mTLS
        v
Repository Jail
        |
        | mTLS
        v
Journal A Jail

Controller
    -> Repository / Journal

AD Authorization Broker / MFA verifier
    -> Active Directory / Domain Controllers

Offline Recovery Authority
    -> recovery environment / Guidon recovery path

FreeBSD Host
    -> Repository Jail + Journal A Jail
```

Mature optional boundaries add:

```text
System 2 / separate Guidon Witness Appliance
    -> Journal B

System 3 / isolated External Witness
    -> write-once advancing Journal checkpoint anchors

System 4 / Recovery Copy Appliance
    -> push-only replication ingress
    -> separate recovery-administration/export trust root
```

These later systems are not implied to share one root, one signing key, or one administrative authority merely because they are all Guidon components.

## Protected endpoint compromise

A compromised endpoint may be able to read data available to the Guidon service, submit malicious or false source content, send malformed protocol data, attempt replay, and attempt abuse of its own credential.

Guidon intends to prevent the endpoint from automatically gaining:

- another endpoint's identity;
- arbitrary Repository filesystem writes;
- Journal signing authority;
- Controller Job signing authority;
- recovery-point deletion authority merely because it performs backups; or
- general remote execution authority.

Guidon does **not** claim that a compromised source cannot submit malicious but internally consistent content. If ransomware encrypts a file and Guidon captures those exact encrypted bytes, Guidon can truthfully identify what it received, from which authenticated endpoint credential, and the resulting hashes. It cannot label the content clean merely because its hash verifies.

Response: `PREVENT + RECORD`, with source-content trustworthiness itself `OUTSIDE CLAIM` unless a defined independent validation establishes more.

## Stolen endpoint transport key

Possession of an endpoint private key is serious. The certificate remains bound to the endpoint UUID and allowed protocol scope; it does not become delete, Journal-sign, Controller-sign, Recovery-Authority, Recovery Copy export, or general Guidon administration power.

Revoked/expired/identity-mismatched credentials are rejected according to policy and the exact reason is recorded.

Response: `PREVENT + RECORD` for cross-authority escalation; stolen-key authenticity within the stolen credential's valid scope remains a real compromise.

## Compromised Controller

Controller compromise, including loss of the Controller Job signing key, is serious but must not automatically grant:

- Journal attestation signing;
- AD Authorization Broker signing;
- Recovery Authority signing;
- endpoint private keys;
- direct Repository filesystem authority;
- Recovery Copy recovery-export administration; or
- External Witness history-management authority.

A normal privileged manual operation still requires the applicable independent user authorization/MFA path, and destructive Repository operations remain subject to Repository validation and Journal gating.

Response: limit blast radius through independent authorities; do not describe Controller compromise as harmless.

## Compromised AD Authorization Broker or MFA verifier

The Broker has narrow authority to establish defined AD authorization facts and sign short-lived exact-Job-bound assertions. An MFA verifier has narrow authority to establish the configured MFA fact. These roles may share one implementation only if the combined authority is explicit.

They do not issue arbitrary Jobs, manipulate backup bytes, perform restores, or sign Journal history.

A compromised Broker/MFA verifier or signing key is a serious authorization-boundary compromise, but assertion scope/expiry/job binding reduce reuse across unrelated operations.

Response: `PREVENT` cross-authority escalation; `RECORD` key/security lifecycle facts where observed.

## TOTP theft/replay

An observed TOTP value is short-lived but may be usable during its accepted time window if replay protection is absent.

Guidon binds successful MFA to one exact Job authorization and records consumed time-step/replay facts without storing the OTP value. The same accepted token/time step must not authorize a second privileged Job under the v1 replay policy.

TOTP values/shared secrets are never intentionally written into Record v1, Journal, Job bodies, logs, support bundles, or crash artifacts.

Response: `PREVENT + RECORD + FAIL CLOSED` for detected replay/invalid/unavailable MFA.

Compromise of the TOTP shared secret itself is a real authentication-factor compromise and is not made harmless by the 30-second timestep.

## Offline theft of Job/control storage

Guidon Jobs are signed exact authorization artifacts and may reveal privileged operation details if stored in plaintext.

Durable Guidon-controlled Job queues/spools use application-level authenticated encryption under `JOB-STORAGE-AND-MFA-V1.md`, with component-specific Job-storage key boundaries.

Response: `PREVENT` plaintext disclosure within the stated offline/key boundary plus `DETECT/FAIL CLOSED` for modified encrypted envelopes.

If an attacker also possesses the required Job-storage KEK, or controls the live privileged OS/process while keys/plaintext are available, confidentiality is not claimed.

## Compromised Repository Jail

A compromised Repository Jail may be able to read/alter/delete Repository-accessible data and attempt to forge local records/state.

It does not possess the Journal signing key. Previously Journal-attested exact Record v1 hashes can therefore conflict with modified Repository bytes and expose tampering within the Journal trust boundary.

Response: `DETECT + RECORD` for covered record-history tampering; content loss itself may still be destructive.

## Compromised Journal Jail

The Journal owns a sensitive attestation signing key. A compromised Journal may be able to forge new attestations while that key remains usable.

The Journal does not receive recovery execution, backup-object browsing, Controller signing, AD authorization, Recovery Authority, or Recovery Copy export powers merely because it is a Journal.

Journal signatures provide facts within the Journal signing-key trust boundary; they are not proof against compromise of that signing authority itself.

Response: blast-radius separation plus optional independent Journal witness/external anchoring. Full signing-authority compromise is not magically solved by its own signatures.

## Single operational Journal unavailable

In Phase 1, a required Journal outage may allow durable backup ingestion to continue but blocks Journal-gated transitions such as recovery-point commit or other defined protected actions.

Response: `FAIL CLOSED + RECORD`.

A mature two-Journal deployment may use an explicitly configured 1-of-2 normal availability policy while visibly reporting degraded witness redundancy. This improves availability but does not make a single surviving witness equivalent to two uncompromised witnesses for compromise resistance.

## Journal witness disagreement

With multiple operational Journals, the same authoritative `record_id` should correlate to the same exact `record_sha256`.

If independent witnesses present:

```text
same record_id + different record_sha256
```

Guidon treats that as a `JOURNAL_WITNESS_CONFLICT`, not normal replication lag.

Response: `DETECT + RECORD`, with stronger protected operations failing closed where policy requires consistent witness state.

## External Witness compromise or rollback

The External Witness is intended to preserve a write-once, always-advancing historical anchor for sealed Journal checkpoints outside operational Guidon systems.

Its API does not normally permit delete/edit/head reset/rollback. COW/append-oriented storage supports this but is not equivalent to hardware WORM.

A malicious root on the External Witness may potentially replace software, delete/destroy local storage, access locally available signing keys, or restore an older internally valid snapshot.

A pure hash chain does not by itself prove that the entire witness has not been rolled back to an older valid head.

Response:

```text
PREVENT/DETECT within the application protocol and available independent high-water-mark controls
OUTSIDE CLAIM for malicious witness-host root rollback until rollback-resistant external/hardware state is implemented and tested
```

## FreeBSD host-root compromise

Host root controls Jails and may inspect/alter local storage, service configuration, binaries, process memory, and locally held key material depending on protection.

Jail separation is meaningful against component compromise but **not** a claimed security boundary against complete malicious host root.

On a primary Guidon appliance, complete host root may therefore threaten both Repository and Journal A if they reside on that host.

A separately administered Journal B, External Witness, or Recovery Copy appliance is not automatically compromised merely because System 1 root was lost.

Response: `OUTSIDE CLAIM` for isolation of components sharing the compromised host; independent systems provide separate contradiction/recovery domains where actually deployed.

## Recovery Copy replication credential compromise

The primary Guidon replication credential is intentionally narrow.

Possession of that credential may permit submission of allowed new immutable recovery material according to Recovery Copy policy, but must not automatically grant:

```text
read/download retained recovery content
delete retained recovery content
modify existing immutable content
Recovery Copy administrator access
Recovery Export creation/download
generic execution
```

Response: `PREVENT + RECORD` for cross-authority escalation.

If the narrow write credential can submit malicious-but-internally-consistent new content, the Recovery Copy records/verifies exactly what it received; it does not claim the source was clean merely because object hashes match.

## Recovery Copy administrator credential compromise

Recovery Copy export administration uses a separate trust root and an administrator credential under the active approved Recovery Copy administration profile, plus required MFA. The initial classical profile uses RSA-4096.

Compromise of that credential/MFA domain is serious because it may expose independently retained recovery data through allowed export operations.

It does not automatically grant primary Repository/Journals/Controller signing authority.

Response: scope separation, exact export authorization, MFA, Records/Journal facts where available, and explicit revocation/rotation procedures. Complete compromise of the Recovery Copy administrator authority remains a real compromise.

The threat remains the same across future credential algorithms; changing the credential profile does not change the Recovery Copy administrator authority or its blast radius.

## Recovery Copy host-root compromise

Root/complete OS compromise of the Recovery Copy appliance may allow an attacker to read mounted/decrypted recovery data, interfere with the push-only API, destroy local copies, inspect process memory, and access locally available keys.

The fact that the primary cannot remotely delete the copy does not protect the copy from its own malicious host root.

Response: `OUTSIDE CLAIM` for local root protection, while independent-system placement still limits propagation from primary-appliance compromise.

## Storage corruption and power loss

Threats include bit corruption, controller/device faults, partial writes, pool problems, abrupt shutdown, kernel panic, forced reset, and uncertain shutdown state.

Guidon responds by beginning a new verification epoch and re-verifying all retained unique objects/manifests/recovery references/required attestations before presenting old recovery points as currently verified.

Response: `DETECT + RECOVER + RECORD`.

Guidon does not claim storage corruption is impossible.

## Network attacker

Assume an attacker can capture traffic, inject/reset packets, replay traffic, attempt plaintext fallback, spoof IP/MAC addresses, and present unauthorized certificates.

Guidon-controlled connections require encryption, peer authentication, identity binding, fail-closed behavior, and no plaintext fallback. Signed Jobs/nonces/expiry provide additional application-level replay protection.

Response: `PREVENT + RECORD`, validated by PCAP/Wireshark acceptance testing.

## Spoofed network observations

IP, MAC, hostname, VLAN, and interface observations are factual context, not cryptographic identity. Guidon does not infer physical user location, gateway/router role, or endpoint authenticity from those observations alone.

Response: `RECORD`; topology/security conclusion from those observations alone is `OUTSIDE CLAIM`.

## Malicious authorized administrator

Guidon cannot prevent every harmful action performed by a legitimately authorized operator with valid required authority.

Its security objective is to make meaningful destructive/security-sensitive operations explicit, constrained, MFA-protected where policy requires, independently gated where defined, and historically attributable rather than silent.

Response: `RECORD + FAIL CLOSED` when a required independent authority is missing. "Authorized admins can never delete backups" is not a claim.

## Ransomware on protected endpoint

Endpoint ransomware may encrypt/delete source data, disable local services, and interfere with the endpoint agent.

Older committed Repository recovery points remain Repository-controlled, and the normal endpoint backup identity must not inherently possess recovery-point deletion authority.

Where an independent Recovery Copy exists, the normal primary replication identity must not possess remote deletion authority over that copy.

Response: `PREVENT` automatic source-to-repository/recovery-copy deletion authority; malicious source content remains subject to the endpoint-compromise limitation above.

## Control-plane compromise

If an attacker obtains every authority needed to perform an operation, Guidon cannot magically distinguish stolen valid authority from legitimate authority.

The architecture reduces shared authority, key reuse, remote deletion capability, and silent actions; it does not claim invulnerability after compromise of all relevant trust anchors/credentials/hosts.

Response: compartmentalization and immutable factual history, with total-authority compromise `OUTSIDE CLAIM`.

## Recovery Authority compromise

The offline Recovery Authority is intentionally powerful for break-glass recovery. Its private key remains outside normal Guidon infrastructure.

Recovery Authority power is scoped to recovery and does not inherently include Repository wipe, Recovery Copy wipe/export administration, CA-trust administration, Journal-key administration, or history deletion.

Response: `PREVENT` cross-authority escalation through scope separation; private-key compromise remains serious.

## Journal signing-key compromise or loss

Key loss and key compromise are different conditions.

Loss prevents future signatures with that key but does not invalidate already verifiable historical artifacts.

Compromise means unauthorized signatures may be possible for the affected key/time window. Guidon records what is known/suspected and begins a new key generation without rewriting old history or manufacturing continuity.

An independent Journal witness or previously anchored external checkpoint may provide additional contradiction/history facts, but neither retroactively proves that all signatures from a compromised key were legitimate.

Response: `RECORD + RECOVER` through explicit key-generation transition; historic trust interpretation remains bounded by what can actually be established.

## CA compromise

Guidon preserves exact leaf/chain certificate hashes and validation times so later review can identify which connections involved an affected CA/chain.

A certificate being valid at the time does not make future CA-compromise discoveries impossible.

Separate Recovery Copy administrative PKI limits automatic propagation of a normal Guidon CA compromise into Recovery Copy export authority.

Response: `RECORD + DETECT` through preserved certificate provenance where a later comparison is performed.

## Catalog compromise/loss

The catalog is not authoritative recovery truth. Valid Repository artifacts can rebuild the catalog; a catalog claim that conflicts with Repository commit artifacts is reported as a mismatch.

A valid Recovery Copy export can also serve as recovery input to a clean Guidon under the Recovery Copy architecture, with independent verification before use.

Response: `RECOVER + DETECT`.

## Record deletion/modification/reordering

Repository record SHA-256 plus Journal entry chaining, receipts, and signed checkpoints are designed to expose covered modifications/deletions/reordering within the Journal trust boundary.

Independent Journal witnesses and External Witness anchoring may increase the number of independently controlled contradiction points.

The accurate security property is **tamper-evident**, not tamper-impossible, especially against complete host-root or signing-key compromise.

Response: `DETECT + RECORD`.

## Explicit unsupported claims

Guidon does not currently claim to:

- survive complete compromise of every Guidon authority, host, witness, recovery copy, and relevant key;
- make malicious-but-internally-consistent source content trustworthy;
- prove physical user/device location from IP/MAC data;
- prove a system is broadly "healthy" without defined validation checks;
- protect Repository and Journal A from complete FreeBSD host-root compromise using Jails alone;
- protect a Recovery Copy or External Witness from malicious root on that same appliance merely because its API is narrow/COW;
- make COW storage equivalent to hardware WORM;
- detect full External Witness rollback without an independently protected high-water-mark/rollback mechanism;
- guarantee stored bytes can never suffer physical/storage corruption;
- infer missing historical events;
- reconstruct exact global event order from unsynchronized distributed timestamps;
- prove workload consistency without a defined workload-specific verification/validation action;
- require Active Directory to remain available during disaster recovery; or
- turn possession of valid cryptographic credentials/MFA factors into proof of human intent.
