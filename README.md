# Secure Multi-Site Enterprise VPN

> Multi-site enterprise infrastructure lab integrating pfSense IPsec, Windows Server, Active Directory and centralized business services across three locations.

<p align="center">
  <img src="assets/architecture/enterprise-multisite-topology.png"
       alt="Conceptual multi-site enterprise network topology"
       width="100%">
</p>

## Overview

This project designs, implements and validates a secure multi-site network for a fictional organization with a headquarters in Lisbon and branch offices in Porto and Faro.

The solution was developed in two complementary layers:

- an **enterprise reference architecture**, covering segmentation, addressing, access policies, services and infrastructure requirements;
- a **functional VMware laboratory**, used to implement and validate the critical network and service dependencies of the proposed design.

Lisbon operates as the central hub and hosts the corporate services. Porto and Faro connect to headquarters through **pfSense IPsec Site-to-Site VPNs**, with inter-branch communication routed through Lisbon using a hub-and-spoke topology.

## Project Scope

The laboratory integrates:

- pfSense firewalls and gateways
- IKEv2 / IPsec Site-to-Site VPN
- Hub-and-spoke multi-site connectivity
- Windows Server
- Active Directory Domain Services
- DNS
- DHCP
- IIS
- MariaDB
- SMB file sharing
- Windows client endpoints
- Firewall policy and least-privilege access
- Structured troubleshooting
- Backup and rollback planning
- Technical intervention cost estimation

## Architecture

The enterprise scenario represents **41 users across three locations**:

| Location | Role | Users |
|---|---|---:|
| Lisbon | Headquarters and central services hub | 25 |
| Porto | Remote branch | 10 |
| Faro | Remote multifunction branch | 6 |

### Laboratory Networks

| Site | LAN | Gateway |
|---|---|---|
| Lisbon | `192.168.10.0/24` | `192.168.10.1` |
| Porto | `192.168.20.0/24` | `192.168.20.1` |
| Faro | `192.168.30.0/24` | `192.168.30.1` |

A dedicated VMware network was used as the simulated WAN transport between the three pfSense firewalls.

## IPsec Design & Final State

The VPN architecture uses **Lisbon as the central hub**, with Site-to-Site IPsec tunnels connecting the Porto and Faro branches to headquarters.

The implementation uses:

- **IKEv2** for Phase 1 negotiation
- Pre-shared key authentication
- AES-256 encryption
- SHA-256 integrity
- Dedicated Phase 2 selectors for each protected LAN
- Additional Phase 2 selectors for **Porto ↔ Faro inter-branch transit through Lisbon**

<p align="center">
  <img src="assets/vpn/ipsec-final-status.png"
       alt="Final pfSense IPsec status showing Lisbon-Porto and Lisbon-Faro tunnels"
       width="100%">
</p>

The final pfSense status shows both Phase 1 connections in **Established** state and the required Phase 2 Security Associations in **Installed** state.

Traffic counters provide additional evidence that the tunnels were actively transporting traffic during validation.

> **What this proves:** IPsec negotiation succeeded and the required Security Associations were installed for the three-site topology.
>
> **What this does not prove by itself:** that DNS, HTTP, SMB, MariaDB or Active Directory are operational across the VPN. Those services were validated separately at the application layer.
>
> ## Application-Layer Validation

Establishing an IPsec tunnel was not considered sufficient proof that the environment was operational. Business services were validated separately from the remote branches.

### MariaDB

From the Porto client, connectivity to the centralized MariaDB service in Lisbon was validated at the application layer by executing a real SQL query:

```sql
SELECT * FROM techsolutions.equipamentos;
```

<p align="center">
  <img src="assets/validation/mariadb-remote-query.png"
       alt="Remote MariaDB query executed from the Porto branch"
       width="90%">
</p>

The query successfully returned records from the database hosted in Lisbon.

> **What this proves:** the remote client could establish a functional MariaDB session through the VPN and execute SQL against the centralized database.
>
> **What this does not prove by itself:** the availability of the other centralized services or the overall health of the VPN. DNS, IIS, SMB and Active Directory were validated independently.

### IIS via Internal DNS

The internal web service hosted in Lisbon was validated from the Porto branch using the corporate DNS name:

`http://portal.techsolutions.local`

<p align="center">
  <img src="assets/validation/iis-internal-dns-validation.png"
       alt="IIS accessed from the Porto branch using the internal DNS name portal.techsolutions.local"
       width="90%">
</p>

Loading the IIS page by hostname validates more than basic HTTP connectivity: the remote client must first resolve the internal DNS name and then successfully reach the web service through the VPN and firewall policy.

> **What this proves:** internal DNS resolution, VPN routing, firewall access and IIS availability were functional for this request.
>
> **What this does not prove by itself:** HTTPS security or certificate validation. This laboratory test intentionally used HTTP.
>
> ### SMB File Service

The centralized SMB file service hosted in Lisbon was validated from the Porto branch by accessing the corporate share directly:

`\\192.168.10.82\PartilhaTechSolutions`

<p align="center">
  <img src="assets/validation/smb-remote-share-validation.png"
       alt="SMB share hosted in Lisbon accessed from the Porto branch"
       width="90%">
</p>

The remote client successfully opened the share and accessed the `LEIA-ME` file.

> **What this proves:** SMB was usable at the application level across the VPN, not merely reachable on TCP port 445.
>
> **What this does not prove by itself:** that domain authentication and the computer trust relationship were healthy. Active Directory was validated separately.

### Active Directory Secure Channel

The Porto client was also validated as an active member of the centralized Active Directory domain.

```powershell
Test-ComputerSecureChannel -Verbose
```

<p align="center">
  <img src="assets/validation/ad-secure-channel-validation.png"
       alt="Successful Active Directory secure channel validation from the Porto client"
       width="90%">
</p>

The command returned `True`, confirming that the secure channel between `CLIENTE-PORTO` and `techsolutions.local` was in good condition.

> **What this proves:** the remote Windows client maintained a functional trust relationship with the Active Directory domain across the multi-site infrastructure.
>
> **What this does not prove by itself:** that every Active Directory-dependent protocol or service is available under every firewall condition. The firewall dependencies were investigated separately during troubleshooting.

## Key Outcomes

- Established IPsec connectivity between **Lisbon ↔ Porto**
- Established IPsec connectivity between **Lisbon ↔ Faro**
- Implemented **Porto ↔ Faro transit through Lisbon**
- Centralized identity and network services in Lisbon
- Joined remote Windows clients to the corporate Active Directory domain
- Validated internal DNS resolution across the VPN
- Validated IIS access using an internal DNS name
- Executed real MariaDB queries from a remote branch
- Accessed SMB resources across the VPN
- Applied service-specific firewall rules instead of unrestricted access
- Diagnosed and corrected multiple controlled network failures
- Repaired an additional Active Directory secure-channel failure
- Defined rollback and change-management procedures
- Produced a technical cost estimate for the intervention

## Validation Approach

Validation was performed progressively by layer:

**Endpoint configuration → Local gateway → IP connectivity → Routing → VPN → Firewall → DNS → Application service**

This approach makes it possible to distinguish between a local endpoint problem, an IPsec failure, a firewall policy issue and an application-layer failure.

## Troubleshooting Scenarios

Six diagnostic scenarios were documented:

| Incident | Root Cause | Final Result |
|---|---|---|
| IPsec authentication failure | Mismatched pre-shared key | VPN restored |
| Phase 2 failure | Incorrect remote network selector | Traffic restored |
| Services unavailable through active VPN | Incomplete firewall policy | Service-specific rules implemented |
| Internal names not resolving | Client using public DNS | Corporate DNS restored |
| Remote networks unreachable | Incorrect default gateway | Routing restored |
| AD secure channel broken | Required AD traffic blocked | Least-privilege AD rules + channel repair |

Each scenario follows the same diagnostic model:

**Known-good baseline → Symptom → Evidence → Hypothesis → Root cause → Correction → Retest**

## Troubleshooting Evidence

### Incident 1 — IPsec Authentication Failure

A controlled authentication failure was introduced by configuring different pre-shared keys on the Lisbon and Porto IPsec peers.

The objective was to determine whether the resulting symptoms could be distinguished from WAN reachability, firewall or Phase 2 configuration problems.

#### Symptom

The IPsec tunnel could not complete Phase 1 negotiation.

The pfSense IPsec logs showed that the peers were able to exchange IKE traffic, but authentication failed during the IKE_AUTH stage.

<p align="center">
  <img src="assets/troubleshooting/ipsec-psk-authentication-failure.png"
       alt="pfSense IPsec logs showing AUTHENTICATION_FAILED during IKE authentication"
       width="95%">
</p>

The relevant log evidence included:

`AUTHENTICATION_FAILED`

This was important because the peers were already exchanging packets. The evidence therefore pointed to an authentication problem rather than a basic WAN connectivity failure.

#### Root Cause

The pre-shared key configured on the Lisbon firewall did not match the key configured on the Porto firewall.

Because the PSK is used during IKE authentication, the mismatch prevented Phase 1 from completing successfully.

#### Correction

The pre-shared key was made identical on both IPsec peers.

The key itself is intentionally excluded from this repository.

#### VPN State After Correction

After correcting the authentication configuration, the Lisbon-Porto tunnel successfully negotiated again.

<p align="center">
  <img src="assets/troubleshooting/ipsec-psk-recovery.png"
       alt="pfSense IPsec tunnel recovered after correcting the pre-shared key"
       width="95%">
</p>

The recovered state showed:

- Phase 1: **Established**
- Phase 2: **Installed**

This confirmed that IKE authentication and the required traffic Security Association had been restored.

#### Functional Validation

Tunnel state alone was not considered sufficient evidence of recovery. Connectivity between the protected networks was tested again after the correction.

<p align="center">
  <img src="assets/troubleshooting/ipsec-psk-connectivity-validation.png"
       alt="Successful connectivity test after recovering the IPsec tunnel"
       width="80%">
</p>

The final test completed with **0% packet loss**, confirming that traffic between the sites was flowing again.

#### Diagnostic Conclusion

| Stage | Evidence |
|---|---|
| Initial state | Known-good VPN baseline |
| Symptom | Phase 1 could not establish |
| Evidence | `AUTHENTICATION_FAILED` in IPsec logs |
| Root cause | Mismatched pre-shared key |
| Correction | Matching PSK restored on both peers |
| Technical validation | Phase 1 `Established`, Phase 2 `Installed` |
| Functional validation | Inter-site connectivity restored with 0% packet loss |

**Key lesson:** successful communication between the VPN peers does not prove successful IPsec authentication. IKE logs were required to distinguish a PSK mismatch from a transport or routing problem.

### Incident 2 — Phase 1 Established but Phase 2 Misconfigured

A second controlled failure was introduced by changing the remote network selector for the Lisbon-Porto Phase 2 association from the correct Porto network `192.168.20.0/24` to `192.168.30.0/24`.

This scenario was designed to demonstrate that an established IKE session does not guarantee functional connectivity between the protected LANs.

#### Symptom

The Lisbon-Porto Phase 1 remained **Established**, but the Phase 2 association responsible for transporting Porto traffic was no longer operational.

<p align="center">
  <img src="assets/troubleshooting/phase2-selector-failure-state.png"
       alt="IPsec Phase 1 established while Phase 2 contains an incorrect remote network selector"
       width="95%">
</p>

The incorrect selector referenced `192.168.30.0/24` instead of the Porto LAN `192.168.20.0/24`.

#### Functional Impact

Although IKE negotiation remained active, traffic between Porto and Lisbon could not use the expected IPsec Security Association.

<p align="center">
  <img src="assets/troubleshooting/phase2-selector-ping-failure.png"
       alt="Connectivity failure caused by an incorrect IPsec Phase 2 selector"
       width="80%">
</p>

The connectivity test resulted in **100% packet loss**.

This distinction is important: the VPN could appear partially healthy because Phase 1 was still established, while the actual protected traffic was unable to reach the remote LAN.

#### Root Cause

The Phase 2 remote network selector was configured as:

`192.168.30.0/24`

instead of the correct Porto network:

`192.168.20.0/24`

The Phase 1 Security Association therefore remained valid, but the Phase 2 traffic selector did not match the network that needed to be protected.

#### Correction

The remote network selector was restored to `192.168.20.0/24`.

After renegotiation, the required Phase 2 Security Association returned to **Installed** state.

<p align="center">
  <img src="assets/troubleshooting/phase2-selector-recovery.png"
       alt="Phase 2 Security Association restored after correcting the remote network selector"
       width="95%">
</p>

#### Functional Validation

Connectivity between the sites was tested again after the selector was corrected.

<p align="center">
  <img src="assets/troubleshooting/phase2-selector-connectivity-validation.png"
       alt="Successful connectivity test after correcting the IPsec Phase 2 selector"
       width="80%">
</p>

The final test completed with **0% packet loss**, confirming that protected traffic was again being transported correctly between the two LANs.

#### Diagnostic Conclusion

| Stage | Evidence |
|---|---|
| Initial condition | Phase 1 remained `Established` |
| Symptom | Porto-Lisbon traffic failed |
| Evidence | Phase 2 referenced the wrong remote LAN |
| Root cause | Incorrect traffic selector |
| Correction | Remote network restored to `192.168.20.0/24` |
| Technical validation | Phase 2 returned to `Installed` |
| Functional validation | Connectivity restored with 0% packet loss |

**Key lesson:** an IPsec VPN showing Phase 1 as `Established` does not prove that application or even IP traffic can cross the tunnel. Phase 2 selectors and data-plane validation must be checked independently.

## Project Status

**Completed laboratory implementation and validation.**

The final environment successfully demonstrated multi-site connectivity and remote access to centralized **Active Directory, DNS, IIS, MariaDB and SMB services**.

---

**Note:** This project was developed in an authorized laboratory environment for training and portfolio purposes. Credentials and other sensitive configuration values are intentionally excluded from the public documentation.
