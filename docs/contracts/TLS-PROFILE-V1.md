# Guidon TLS Profile v1

## Purpose

This contract freezes the minimum transport-security behavior required for the first Guidon network services so different components do not make incompatible or silently weaker TLS decisions.

## Governing rule

Every Guidon-controlled connection carrying data, control, authentication, Journal, recovery, management, or integration traffic is:

```text
encrypted
peer-authenticated
identity-bound
fail-closed
no plaintext fallback
```

The network-security architecture remains authoritative for factual observation and PCAP/Wireshark acceptance requirements. This profile defines the v1 implementation floor.

## TLS version

Guidon TLS Profile v1 requires:

```text
TLS 1.3 preferred
TLS 1.2 permitted when required for supported platform interoperability
TLS 1.1 and earlier prohibited
SSL prohibited
```

A connection must not retry using a lower protocol version below the configured supported floor merely because negotiation failed.

## Cipher suites and key exchange

For TLS 1.3, use the platform TLS implementation's supported AEAD TLS 1.3 suites without enabling deprecated compatibility ciphers.

For TLS 1.2, only suites providing all of the following are permitted:

```text
AEAD
forward secrecy through ephemeral ECDHE
certificate-authenticated peer identity
```

Prohibited TLS 1.2 properties include:

```text
RC4
3DES
DES
NULL encryption
EXPORT suites
static RSA key exchange
CBC-only suites for Guidon application traffic
anonymous authentication
```

The exact platform cipher list is implementation/configuration data and may evolve within this profile's security properties.

## Mutual TLS

Connections that the architecture marks as mTLS require valid certificates from both peers.

Absence of a required client certificate is a connection rejection, not a reason to continue with server-only TLS.

## Certificate chain validation

Each peer validates at least:

- certificate validity period;
- chain construction to an explicitly configured Guidon trust anchor;
- signature validity;
- required key usage / extended key usage;
- revocation status according to the applicable v1 revocation policy;
- cryptographic key strength supported by the deployment profile; and
- Guidon identity binding in addition to ordinary PKI chain validity.

A certificate that chains to an approved CA is not sufficient by itself to represent an arbitrary Guidon endpoint/component identity.

## Identity binding

The authenticated certificate is bound to a stable Guidon identity.

For protected endpoints this is the stable `endpoint_id` UUID.

For infrastructure services it is the stable Guidon component/service identity defined by configuration/trust contracts.

The implementation must validate the claimed Guidon identity against the exact certificate credential authorized for that identity. Hostname/IP observations alone do not replace this binding.

The exact X.509 encoding used for Guidon identity binding must be frozen in the implementation schema before production use and acceptance-tested for mismatch behavior.

## Certificate algorithms

TLS Profile v1 permits certificate/public-key algorithms supported securely by the target Windows and FreeBSD platforms, with the initial preferred direction:

```text
ECDSA P-256 or stronger supported curve
or
RSA 3072-bit or stronger
```

RSA 2048 may be accepted only where required for supported enterprise PKI interoperability during pre-alpha/pilot compatibility testing and must be explicitly visible in configuration/observations rather than silently treated as the preferred profile.

Weak/legacy algorithms such as MD5-signed certificates are prohibited. SHA-1 certificate signatures are not accepted for newly issued Guidon identities.

## Revocation

Where the configured PKI supports revocation, Guidon performs revocation checking using the defined enterprise mechanism, such as CRL and/or OCSP as supported by the platform/deployment.

For a security-sensitive or privileged Guidon connection requiring revocation assurance:

```text
revoked
    -> reject

revocation status cannot be established when policy requires it
    -> fail closed / reject or mark not_determined according to the exact connection contract
```

Guidon must not silently disable revocation checking because the responder/distribution point is unavailable.

The exact deployment policy states whether a particular connection class requires hard-fail revocation checking. That policy generation is preserved as configuration provenance.

## Certificate rotation

Stable Guidon identity is separate from renewable credentials.

A controlled rotation may authorize a bounded overlap of old/new certificates for the same identity. Every accepted connection records the exact certificate actually presented.

Removing/adding an authorized certificate binding is a security-sensitive configuration/trust change and follows the defined Journal-gated change path.

## Plaintext rejection

Listeners intended for TLS do not provide an HTTP/plaintext compatibility endpoint on the same Guidon application path.

A plaintext connection attempt is rejected and, where sufficient protocol state exists to do so safely, recorded as an observed connection rejection.

Guidon does not redirect plaintext HTTP to HTTPS as a substitute for fail-closed transport requirements.

## Timeouts and resource bounds

Every network listener/client uses bounded:

```text
connection establishment time
TLS handshake time
request header/artifact time
idle time
maximum request/artifact size
maximum concurrent work according to implementation limits
```

A peer cannot hold an unbounded pre-authentication connection or force unbounded parser allocation.

Exact values are implementation configuration, not architecture constants, and are preserved through configuration provenance.

## Session resumption / early data

TLS 1.3 0-RTT early data is disabled for Guidon state-changing or authority-bearing requests in v1.

Guidon must not allow replayable TLS early data to bypass Job v1 nonce/replay handling or Journal/Repository gating semantics.

Session resumption may be used only where peer identity and current authorization/trust requirements remain correctly revalidated by the selected TLS implementation/profile.

## Regulated cryptographic deployment profiles

This TLS profile defines protocol/security behavior; `../architecture/CRYPTOGRAPHIC-ARCHITECTURE.md` defines the higher-level cryptographic compliance/profile rules.

Where a deployment requires a FIPS 140-3 validated cryptographic implementation or another regulated cryptographic boundary, the TLS implementation must use an allowed provider/module and operating mode covered by that deployment profile. A configured/requested FIPS mode is not by itself proof that the required validated module/provider is actually in use.

Guidon records/configures the applicable provider/module/profile provenance where it can establish it. If the required regulated TLS profile cannot be established, Guidon rejects the connection/operation rather than silently negotiating through a nonconforming provider or weaker profile.

## Private-key handling

Transport private keys are separate from:

```text
Journal Ed25519 attestation keys
Controller Job signing keys
AD Authorization Broker signing keys
Recovery Authority keys
```

Endpoint transport private keys should normally be generated/retained on the endpoint and not exported to the Repository.

## Acceptance tests

Every implemented Guidon-controlled connection type must test at least:

- successful required TLS/mTLS path;
- TLS 1.1/legacy protocol attempt;
- plaintext attempt;
- missing required client certificate;
- wrong Guidon identity with otherwise trusted certificate;
- untrusted CA;
- expired/not-yet-valid certificate;
- revoked certificate where revocation applies;
- unavailable revocation source under hard-fail policy;
- unauthorized certificate rotation/binding;
- malformed TLS traffic/resource limits;
- disabled certificate validation regression; and
- black-box PCAP showing no Guidon application payload in cleartext.

In a controlled lab, session-key logging may be used to decrypt a test PCAP and verify the intended application protocol. Production systems must not routinely log TLS session secrets.
