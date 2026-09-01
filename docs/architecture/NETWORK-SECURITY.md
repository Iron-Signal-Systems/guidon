# Network Security Architecture

## Governing requirement

Every Guidon-controlled connection carrying data, control, authentication, Journal, recovery, management, or integration traffic must be:

```text
encrypted
peer-authenticated
identity-bound
fail-closed
no plaintext fallback
```

This applies to traffic between separate systems and to Guidon Jails sharing one FreeBSD host.

## Protected endpoint direction

The normal Windows protection path is outbound from the protected endpoint:

```text
Protected Windows Endpoint
        |
        | outbound TCP 443 / mTLS
        v
Guidon Repository
```

Guidon should not require broad inbound Repository-to-endpoint access for ordinary backup operation.

Endpoint certificate identity is bound to the stable Guidon endpoint UUID. A certificate that chains to an approved CA is not accepted merely because the chain is valid; the presented credential must be authorized for the identity it represents.

## Infrastructure connections

Repository-to-Journal, Controller-to-Repository/Journal, Broker connections, recovery connections, and future Guidon integrations follow the same encrypted/authenticated/identity-bound rule.

Sharing a host or internal network does not create an exemption.

## No arbitrary protocol fallback

Guidon must not implement behavior such as:

```text
try mTLS
if that fails, retry HTTP
```

or silently disable certificate validation for availability.

If required transport authentication cannot be established, the affected operation fails closed and the exact condition is recorded.

## Factual network observations

Guidon records network observations as observations, not topology conclusions.

The locked Layer-2 fields are:

```text
layer2_source_mac_observed
layer2_destination_mac_observed
observation_point
interface
vlan_id_if_observed
```

Where observed, factual IP/port/connection fields may also be recorded.

Guidon does **not** automatically label a MAC address as:

```text
gateway
router
server
direct_destination
previous_hop
```

because the meaning depends on where the observation was made and how the network is constructed.

On a same-L2 path, the destination MAC may be the final device. On a routed path, a sender may observe a router/gateway interface MAC. Guidon preserves the observed addresses and observation point; the administrator or a separate correlation system may interpret them.

If another Iron Signal Systems product such as Atlas later correlates MAC/VLAN/switch-port information, the original Guidon observation remains unchanged and the correlation is kept as a separate result.

IP address, MAC address, and hostname are observations, not cryptographic endpoint identity.

`vlan_id_if_observed` is populated only when the VLAN is actually observable at the capture/observation point. An untagged access-port observation may legitimately be `not_observed`.

## Certificate observations

Every certificate-authenticated connection preserves the exact certificate facts defined in the identity/trust contract, including the observed thumbprint/algorithm and Guidon-calculated exact-DER SHA-256.

Connection acceptance/rejection is recorded with the actual validation performed and the exact policy condition that allowed or blocked the connection.

Avoid broad fields such as `trusted: true` when more precise facts are available.

## PCAP and Wireshark acceptance testing

Every implemented Guidon-controlled connection type requires packet-capture acceptance testing.

At minimum test:

- successful encrypted/authenticated connection;
- no certificate where one is required;
- certificate for the wrong Guidon identity;
- untrusted CA;
- expired certificate;
- revoked certificate where revocation testing applies;
- plaintext connection attempt;
- disabled/invalid TLS behavior; and
- replay/tamper cases relevant to the application protocol.

A normal black-box PCAP without session keys must not expose Guidon application payload.

In a controlled lab only, TLS session-key logging may be used to decrypt a test PCAP and prove that the protected protocol contains the expected application data. Production Guidon systems must not routinely log TLS session secrets.

Where technically possible, acceptance testing should compare:

```text
Guidon connection record
endpoint/local certificate observation
Repository/peer certificate observation
PCAP/Wireshark observation
```

The test proves what the implemented path actually does; documentation alone is not sufficient.

## Network attacks in scope

The transport design assumes an attacker may be able to:

- capture traffic;
- inject/reset packets;
- replay captured protocol traffic;
- attempt plaintext connections;
- spoof IP/MAC addresses; and
- present unauthorized certificates.

mTLS and signed application objects are intended to provide defined protection against those actions, but network observations alone are never elevated into proof of user or endpoint identity.
