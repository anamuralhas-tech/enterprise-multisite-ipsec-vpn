# Incident 03 — Incomplete Firewall Policy

## Summary

A controlled failure was introduced by reducing the firewall permissions applied to traffic between the Porto and Lisbon networks.

The IPsec VPN remained established, but ICMP, DNS and SMB became unavailable.

The objective was to demonstrate that VPN establishment and firewall authorization are independent layers and that an operational tunnel does not guarantee that required business services are permitted.

## Environment

| Component | Configuration |
|---|---|
| VPN | Lisbon-Porto site-to-site IPsec |
| Lisbon LAN | `192.168.10.0/24` |
| Porto LAN | `192.168.20.0/24` |
| Central server | `SRV-LISBOA` |
| Internal DNS | `192.168.10.82` |
| SMB service | TCP `445` |
| DNS service | TCP/UDP `53` |

## Known-Good Baseline

Before introducing the fault, the VPN was operational and the Porto client could reach centralized services in Lisbon.

The VPN Security Associations were maintained while firewall permissions were deliberately reduced.

This allowed the firewall policy to be tested independently from tunnel establishment.

## Fault Introduced

The IPsec firewall policy was changed so that it no longer permitted all traffic required between Porto and Lisbon.

The VPN remained established, but service-specific communication became incomplete.

## Symptom 1 — ICMP Failure

The Porto client could no longer receive ICMP replies from the Lisbon network.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-icmp-failure.png"
       alt="ICMP connectivity failure caused by an incomplete IPsec firewall policy"
       width="80%">
</p>

The connectivity test completed with:

**100% packet loss**

This demonstrated that tunnel establishment did not imply that ICMP traffic was authorized.

## Symptom 2 — DNS Timeout

Internal DNS resolution also failed.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-dns-timeout.png"
       alt="DNS timeout caused by an incomplete IPsec firewall policy"
       width="80%">
</p>

The internal query timed out because the required DNS traffic was not permitted through the policy.

## Symptom 3 — SMB Unavailable

The SMB service was independently tested using TCP port `445`.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-smb-445-failure.png"
       alt="SMB TCP port 445 inaccessible because of the firewall policy"
       width="80%">
</p>

`Test-NetConnection` showed that TCP port `445` was not reachable from Porto.

The combined failures across ICMP, DNS and SMB indicated that the problem was broader than a single application.

## Diagnostic Reasoning

The evidence was evaluated by separating tunnel state from firewall authorization.

| Observation | Interpretation |
|---|---|
| IPsec VPN remained established | Tunnel negotiation was not the primary failure |
| ICMP failed | Traffic policy required investigation |
| DNS queries timed out | Service-specific traffic was being blocked |
| TCP 445 was unreachable | SMB traffic was also denied |
| Multiple services failed simultaneously | Common policy layer became the primary hypothesis |

### What These Tests Proved

The tests proved that several types of traffic crossing the VPN were unavailable even though the tunnel itself remained operational.

### What These Tests Did Not Prove

A successful IPsec state did not prove that the required protocols were authorized by the firewall.

Each service therefore had to be retested independently after the policy was corrected.

## Root Cause

The IPsec firewall policy contained insufficient permissions for the required Porto-Lisbon traffic.

The tunnel could remain operational while the firewall independently denied legitimate inter-site communication.

## Correction

The policy was rebuilt using explicit rules identified by protocol and service.

Specific permissions were created for:

- ICMP;
- DNS;
- SMB;
- HTTP;
- MariaDB.

The final design avoided relying on a broad permanent `Any` rule.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-corrected-rules.png"
       alt="Corrected pfSense IPsec firewall rules separated by protocol and service"
       width="95%">
</p>

This made the permitted traffic easier to audit and aligned the policy with a least-privilege approach.

## ICMP Validation

ICMP connectivity was tested again after applying the corrected rules.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-icmp-recovery.png"
       alt="ICMP connectivity restored after correcting the firewall policy"
       width="80%">
</p>

The ping completed with:

**0% packet loss**

## DNS Validation

Internal DNS resolution was tested independently.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-dns-recovery.png"
       alt="Internal DNS resolution restored after correcting the firewall policy"
       width="80%">
</p>

The hostname:

`portal.techsolutions.local`

successfully resolved to:

`192.168.10.82`

This confirmed that the Porto client could again reach the centralized Lisbon DNS service.

## SMB Validation

TCP port `445` was then retested from Porto.

<p align="center">
  <img src="../assets/troubleshooting/firewall-policy-smb-recovery.png"
       alt="SMB TCP port 445 reachable after correcting the firewall policy"
       width="80%">
</p>

The SMB service became reachable again.

## Incident Timeline

| Stage | Result |
|---|---|
| Baseline | VPN and centralized services operational |
| Fault introduced | Firewall permissions deliberately reduced |
| VPN state | Remained established |
| ICMP symptom | 100% packet loss |
| DNS symptom | Internal queries timed out |
| SMB symptom | TCP 445 unreachable |
| Root cause | Incomplete IPsec firewall policy |
| Correction | Explicit service-specific firewall rules |
| ICMP validation | 0% packet loss |
| DNS validation | `portal.techsolutions.local` resolved to `192.168.10.82` |
| SMB validation | TCP 445 reachable |

## Key Technical Takeaway

An established IPsec tunnel proves that VPN negotiation is operational, but it does not prove that required business traffic is authorized.

Troubleshooting must distinguish between:

- VPN control-plane state;
- firewall authorization;
- individual protocol reachability;
- application-layer availability.

The incident was considered resolved only after the required services were independently retested through the corrected firewall policy.
