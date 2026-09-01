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

Initial boundaries include:

```text
Protected Windows Endpoint
        |
        | mTLS
        v
Repository Jail
        |
        | mTLS
        v
Journal Jail

Controller
    -> Repository / Journal

AD Authorization Broker
    -> Active Directory / Domain Controllers

Offline Recovery Authority
    -> recovery environment / Guidon recovery path

FreeBSD Host
    -> Repository Jail + Journal Jail
```

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

Possession of an endpoint private key is serious. The certificate remains bound to the endpoint UUID and allowed protocol scope; it does not become delete, Journal-sign, Controller-sign, or Recovery-Authority power.

Revoked/expired/identity-mismatched credentials are rejected according to policy and the exact reason is recorded.

Response: `PREVENT + RECORD` for cross-authority escalation; stolen-key authenticity within the stolen credential's valid scope remains a real compromise.

## Compromised Controller

Controller compromise, including loss of the Controller Job signing key, is serious but must not automatically grant:

- Journal attestation signing;
- AD Authorization Broker signing;
- Recovery Authority signing;
- endpoint private keys; or
- direct Repository filesystem authority.

A normal privileged manual operation still requires the applicable independent user authorization path, and destructive Repository operations remain subject to Repository validation and Journal gating.

Response: limit blast radius through independent authorities; do not describe Controller compromise as harmless.

## Compromised AD Authorization Broker

The Broker has narrow authority to establish defined AD authorization facts and sign short-lived Job-bound assertions. It does not issue Jobs, manipulate backup bytes, perform restores, or sign Journal history.

A compromised Broker/signing key is a serious authorization-boundary compromise, but assertion scope/expiry/job binding reduce reuse across unrelated operations.

Response: `PREVENT` cross-authority escalation; `RECORD` key/security lifecycle facts where observed.

## Compromised Repository Jail

A compromised Repository Jail may be able to read/alter/delete Repository-accessible data and attempt to forge local records/state.

It does not possess the Journal signing key. Previously Journal-attested exact Record v1 hashes can therefore conflict with modified Repository bytes and expose tampering within the Journal trust boundary.

Response: `DETECT + RECORD` for covered record-history tampering; content loss itself may still be destructive.

## Compromised Journal Jail

The Journal owns a sensitive attestation signing key. A compromised Journal may be able to forge new attestations while that key remains usable.

The Journal does not receive recovery execution, backup-object browsing, Controller signing, AD authorization, or Recovery Authority powers merely because it is the Journal.

Journal signatures provide evidence within the Journal signing-key trust boundary; they are not proof against compromise of that signing authority itself.

Response: blast-radius separation plus future opportunity for HSM/external anchoring. Full signing-authority compromise is not magically solved by its own signatures.

## FreeBSD host-root compromise

Host root controls Jails and may inspect/alter local storage, service configuration, binaries, and locally held key material depending on protection.

Jail separation is meaningful against component compromise but **not** a claimed security boundary against complete malicious host root.

Response: `OUTSIDE CLAIM` for Jail isolation alone. Future external/hardware-backed mechanisms may strengthen this.

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

Its security objective is to make meaningful destructive/security-sensitive operations explicit, constrained, independently gated where defined, and historically attributable rather than silent.

Response: `RECORD + FAIL CLOSED` when a required independent authority is missing. "Authorized admins can never delete backups" is not a claim.

## Ransomware on protected endpoint

Endpoint ransomware may encrypt/delete source data, disable local services, and interfere with the endpoint agent.

Older committed Repository recovery points remain Repository-controlled, and the normal endpoint backup identity must not inherently possess recovery-point deletion authority.

Response: `PREVENT` automatic source-to-repository deletion authority; malicious source content remains subject to the endpoint-compromise limitation above.

## Control-plane compromise

If an attacker obtains every authority needed to perform an operation, Guidon cannot magically distinguish stolen valid authority from legitimate authority.

The architecture reduces shared authority, key reuse, and silent actions; it does not claim invulnerability after compromise of all relevant trust anchors/credentials.

Response: compartmentalization and immutable factual history, with total-authority compromise `OUTSIDE CLAIM`.

## Recovery Authority compromise

The offline Recovery Authority is intentionally powerful for break-glass recovery. Its private key remains outside normal Guidon infrastructure.

Recovery Authority power is scoped to recovery and does not inherently include Repository wipe, CA-trust administration, Journal-key administration, or history deletion.

Response: `PREVENT` cross-authority escalation through scope separation; private-key compromise remains serious.

## Journal signing-key compromise or loss

Key loss and key compromise are different conditions.

Loss prevents future signatures with that key but does not invalidate already verifiable historical artifacts.

Compromise means unauthorized signatures may be possible for the affected key/time window. Guidon records what is known/suspected and begins a new key generation without rewriting old history or manufacturing continuity.

Response: `RECORD + RECOVER` through explicit key-generation transition; historic trust interpretation remains bounded by what can actually be established.

## CA compromise

Guidon preserves exact leaf/chain certificate hashes and validation times so later review can identify which connections involved an affected CA/chain.

A certificate being valid at the time does not make future CA-compromise discoveries impossible.

Response: `RECORD + DETECT` through preserved certificate provenance where a later comparison is performed.

## Catalog compromise/loss

The catalog is not authoritative recovery truth. Valid Repository artifacts can rebuild the catalog; a catalog claim that conflicts with Repository commit artifacts is reported as a mismatch.

Response: `RECOVER + DETECT`.

## Record deletion/modification/reordering

Repository record SHA-256 plus Journal entry chaining, receipts, and signed checkpoints are designed to expose covered modifications/deletions/reordering within the Journal trust boundary.

The accurate security property is **tamper-evident**, not tamper-impossible, especially against complete host-root or signing-key compromise.

Response: `DETECT + RECORD`.

## Explicit unsupported claims

Guidon does not currently claim to:

- survive complete compromise of every Guidon authority;
- make malicious-but-internally-consistent source content trustworthy;
- prove physical user/device location from IP/MAC data;
- prove a system is broadly "healthy" without defined validation checks;
- protect Repository and Journal from complete FreeBSD host-root compromise using Jails alone;
- guarantee stored bytes can never suffer physical/storage corruption;
- infer missing historical events;
- reconstruct exact global event order from unsynchronized distributed timestamps;
- prove workload consistency without a defined workload-specific verification/validation action;
- require Active Directory to remain available during disaster recovery; or
- turn possession of valid cryptographic credentials into proof of human intent.
