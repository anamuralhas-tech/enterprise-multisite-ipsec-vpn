# Incident 01 — IPsec PSK Authentication Failure

## Summary

A controlled failure was introduced in the Lisbon-Porto site-to-site IPsec VPN by configuring different pre-shared keys on the two peers.

The objective was to reproduce an authentication failure, identify the relevant evidence in the pfSense IPsec logs, correct the configuration and independently validate both tunnel state and inter-site connectivity.

## Environment

| Component | Configuration |
|---|---|
| VPN peers | pfSense Lisbon and pfSense Porto |
| VPN type | Site-to-Site IPsec |
| IKE version | IKEv2 |
| Authentication | Pre-Shared Key |
| Lisbon LAN | `192.168.10.0/24` |
| Porto LAN | `192.168.20.0/24` |
| Lisbon WAN lab address | `192.168.136.10` |
| Porto WAN lab address | `192.168.136.20` |

## Known-Good Baseline

Before introducing the fault, the Lisbon-Porto VPN was operational and inter-site traffic was successfully passing through the tunnel.

A single configuration fault was then introduced so that the resulting symptoms could be attributed to a known change.

## Fault Introduced

The pre-shared key configured on one VPN peer was changed so that it no longer matched the key configured on the opposite peer.

The actual keys are intentionally excluded from this repository.

## Symptom

The Lisbon-Porto VPN could no longer complete IPsec authentication.

The failure occurred during IKE authentication rather than during basic IP transport between the VPN peers.

## Evidence

The pfSense IPsec logs showed that IKE packets were exchanged between the peers, followed by an authentication failure.

<p align="center">
  <img src="../assets/troubleshooting/ipsec-psk-authentication-failure.png"
       alt="pfSense IPsec logs showing AUTHENTICATION_FAILED during IKE authentication"
       width="95%">
</p>

Relevant log evidence included:

`AUTHENTICATION_FAILED`

The presence of transmitted and received IKE packets was diagnostically important.

It demonstrated that the peers were able to communicate at the network layer, making a basic WAN reachability failure less likely.

## Diagnostic Reasoning

The evidence was evaluated progressively.

| Observation | Interpretation |
|---|---|
| VPN previously worked | Established known-good baseline |
| IKE packets were exchanged | Peer-to-peer IP connectivity existed |
| Authentication failed during IKE_AUTH | Failure occurred during authentication |
| `AUTHENTICATION_FAILED` appeared in logs | Authentication parameters required investigation |
| PSKs differed between peers | Root cause identified |

### What This Test Proved

The logs proved that the peers were communicating and that IKE authentication was failing.

### What This Test Did Not Prove

The logs alone did not prove that the protected LAN traffic would function after authentication was repaired.

For that reason, tunnel state and end-to-end connectivity were validated independently after the correction.

## Root Cause

The pre-shared key configured on the Lisbon peer did not match the key configured on the Porto peer.

Because both peers must use the same PSK during IKE authentication, the mismatch prevented Phase 1 from completing.

## Correction

The PSK configuration was made consistent on both VPN peers.

No other VPN parameter was changed as part of the correction.

This preserved the troubleshooting principle of correcting only the identified root cause before retesting.

## Technical Validation

After correcting the authentication configuration, the IPsec tunnel negotiated successfully again.

<p align="center">
  <img src="../assets/troubleshooting/ipsec-psk-recovery.png"
       alt="IPsec tunnel recovered after correcting the pre-shared key"
       width="95%">
</p>

The recovered state showed:

- Phase 1: **Established**
- Phase 2: **Installed**

This demonstrated successful IKE authentication and installation of the required IPsec Security Association.

## Functional Validation

Tunnel status alone was not treated as sufficient proof of service restoration.

Connectivity between the protected networks was therefore tested again.

<p align="center">
  <img src="../assets/troubleshooting/ipsec-psk-connectivity-validation.png"
       alt="Successful inter-site connectivity after recovering the IPsec VPN"
       width="80%">
</p>

The final connectivity test completed with:

**0% packet loss**

This confirmed that traffic between the protected networks was again flowing through the VPN.

## Incident Timeline

| Stage | Result |
|---|---|
| Baseline | VPN operational |
| Fault introduced | PSKs deliberately mismatched |
| Primary symptom | IPsec authentication failed |
| Log evidence | `AUTHENTICATION_FAILED` |
| Root cause | Mismatched PSK |
| Correction | Matching PSK restored |
| VPN validation | Phase 1 `Established`, Phase 2 `Installed` |
| Functional validation | Inter-site connectivity restored with 0% packet loss |

## Key Technical Takeaway

Successful IP communication between VPN peers does not prove successful IPsec authentication.

A failure at the IKE authentication layer can occur even when the peers can exchange packets normally.

Troubleshooting therefore needs to distinguish between:

- transport reachability;
- IKE authentication;
- IPsec Security Association state;
- protected network connectivity.

The incident was considered resolved only after both the control-plane state and end-to-end data-plane connectivity had been validated.
