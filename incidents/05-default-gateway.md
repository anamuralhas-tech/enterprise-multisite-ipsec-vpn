# Incident 05 — Incorrect Default Gateway

## Summary

A controlled failure was introduced by changing the default gateway configured on `CLIENTE-PORTO`.

The correct gateway for the Porto network was `192.168.20.1`, but the client was deliberately configured to use `192.168.20.254`.

The VPN and firewall remained operational, but the endpoint could no longer correctly forward traffic destined for remote networks.

The objective was to distinguish an endpoint routing problem from a VPN, firewall or remote-service failure.

## Environment

| Component | Configuration |
|---|---|
| Client | `CLIENTE-PORTO` |
| Porto LAN | `192.168.20.0/24` |
| Correct default gateway | `192.168.20.1` |
| Deliberately incorrect gateway | `192.168.20.254` |
| Remote server | `SRV-LISBOA` |
| Remote server IP | `192.168.10.82` |

## Known-Good Baseline

Before introducing the fault, `CLIENTE-PORTO` used the Porto pfSense firewall at `192.168.20.1` as its default gateway and could communicate with the remote Lisbon network.

The gateway was then changed while the VPN and firewall infrastructure remained operational.

## Fault Introduced

The client default gateway was deliberately changed from:

`192.168.20.1`

to:

`192.168.20.254`

<p align="center">
  <img src="../assets/troubleshooting/gateway-misconfiguration-wrong-default-route.png"
       alt="CLIENTE-PORTO configured with incorrect default gateway 192.168.20.254"
       width="80%">
</p>

This altered the endpoint routing path without changing the VPN configuration.

## Symptom

Traffic destined for networks outside `192.168.20.0/24` could no longer be forwarded correctly.

Communication with the Lisbon server failed even though the IPsec tunnel and firewall policy remained operational.

## Evidence

A connectivity test toward the remote Lisbon network produced:

`Destination host unreachable`

<p align="center">
  <img src="../assets/troubleshooting/gateway-misconfiguration-connectivity-failure.png"
       alt="Destination host unreachable caused by an incorrect client default gateway"
       width="80%">
</p>

The unreachable message was generated locally by `CLIENTE-PORTO`.

This was diagnostically important because it did not represent a valid response from the remote server.

## Diagnostic Reasoning

The evidence was evaluated by focusing on where the failure occurred in the traffic path.

| Observation | Interpretation |
|---|---|
| VPN remained operational | Tunnel failure was less likely |
| Firewall remained operational | Central policy failure was less likely |
| Remote networks became unreachable from one endpoint | Local routing became a primary hypothesis |
| `Destination host unreachable` was generated locally | Packet forwarding failed before reaching the remote destination |
| Default gateway was `192.168.20.254` | Incorrect local route identified |
| Correct gateway was `192.168.20.1` | Root cause confirmed |

### What This Test Proved

The local unreachable response proved that the endpoint could not correctly forward traffic toward the remote network.

### What This Test Did Not Prove

The message did not prove a remote-server failure, VPN failure or firewall block.

Because the error was generated locally, the endpoint routing configuration had to be validated first.

## Root Cause

The default gateway configured on `CLIENTE-PORTO` was incorrect.

The client belonged to the `192.168.20.0/24` network and should have used the Porto pfSense firewall:

`192.168.20.1`

Instead, the default route pointed to:

`192.168.20.254`

As a result, packets destined for remote networks could not be routed correctly.

## Correction

The default gateway was restored to:

`192.168.20.1`

<p align="center">
  <img src="../assets/troubleshooting/gateway-default-route-restored.png"
       alt="CLIENTE-PORTO default gateway restored to 192.168.20.1"
       width="80%">
</p>

The client was again configured to use the Porto firewall as its route toward remote networks.

## Functional Validation

Connectivity with the Lisbon server was tested again after restoring the correct gateway.

<p align="center">
  <img src="../assets/troubleshooting/gateway-connectivity-restored.png"
       alt="Connectivity to SRV-LISBOA restored after correcting the default gateway"
       width="80%">
</p>

The final ping to `SRV-LISBOA` completed with:

**0% packet loss**

This confirmed that endpoint routing toward the remote Lisbon network had been restored.

## Incident Timeline

| Stage | Result |
|---|---|
| Baseline | Remote connectivity operational |
| Fault introduced | Default gateway changed to `192.168.20.254` |
| Symptom | Remote network unreachable |
| Evidence | Local `Destination host unreachable` message |
| Root cause | Incorrect endpoint default route |
| Correction | Gateway restored to `192.168.20.1` |
| Functional validation | Ping to `SRV-LISBOA` restored with 0% packet loss |

## Key Technical Takeaway

When VPN and firewall infrastructure are operational but a single endpoint cannot reach remote networks, local routing should be validated before changing tunnel or firewall configuration.

Troubleshooting should distinguish between:

- endpoint IP configuration;
- default route and gateway;
- VPN availability;
- firewall authorization;
- remote-host availability.

A locally generated unreachable message is strong evidence that the failure may occur before traffic ever reaches the VPN gateway.
