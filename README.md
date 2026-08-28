# Secure Multi-Site Enterprise VPN

**Enterprise network infrastructure lab integrating pfSense IPsec, Windows Server, Active Directory and centralized services across three geographically separated sites.**

<p align="center">
  <img src="assets/architecture/enterprise-multisite-topology.png"
       alt="TechSolutions multi-site enterprise network topology"
       width="100%">
</p>

## Overview

This project designs and validates a secure multi-site infrastructure for a fictional 41-user organization distributed across Lisbon, Porto and Faro.

Lisbon operates as the central site and IPsec hub, hosting the shared infrastructure services used by the remote branches. Porto and Faro connect securely to Lisbon through site-to-site IPsec tunnels, with controlled inter-site transit through the central firewall.

The project combines two complementary perspectives:

- an **enterprise infrastructure design**, covering topology, addressing, security policy and service architecture;
- a **functional VMware laboratory**, used to implement, test, troubleshoot and validate the proposed solution.

The laboratory integrates pfSense firewalls, Windows Server, Active Directory Domain Services, DNS, DHCP, IIS, MariaDB and SMB services.

Validation was performed progressively from network connectivity to application-layer functionality. Controlled faults were then introduced to test an evidence-driven troubleshooting methodology, followed by an additional real Active Directory secure channel incident discovered during final validation.

## Project Scope

The project covers the design, implementation and validation of a three-site enterprise infrastructure with centralized services and secure inter-site connectivity.

Core components include:

- pfSense firewalls at Lisbon, Porto and Faro;
- IKEv2 site-to-site IPsec;
- hub-and-spoke topology with Lisbon as the central hub;
- Porto-Faro transit through Lisbon;
- Windows Server infrastructure;
- Active Directory Domain Services;
- internal DNS and DHCP;
- IIS internal web service;
- MariaDB database service;
- SMB file sharing;
- Windows domain clients;
- service-specific firewall policies;
- controlled troubleshooting and root-cause validation;
- rollback and change-management planning;
- technical cost estimation.

## Architecture

### Site Roles

| Location | Role | Users |
|---|---|---:|
| Lisbon | Headquarters, VPN hub and centralized services | 25 |
| Porto | Remote branch using centralized Lisbon services | 10 |
| Faro | Remote branch with inter-site communication through Lisbon | 6 |
| **Total** |  | **41** |

Lisbon acts as the infrastructure core. Active Directory, DNS, DHCP, IIS, MariaDB and SMB are centralized on `SRV-LISBOA`, while Porto and Faro consume those services through the IPsec infrastructure.

The enterprise design and the laboratory implementation are deliberately separated: the enterprise layer represents the proposed business architecture, while the VMware environment reproduces the network and service dependencies required to validate the design.

### Laboratory Networks

| Site / Function | Network | Gateway |
|---|---|---|
| Lisbon LAN | `192.168.10.0/24` | `192.168.10.1` |
| Porto LAN | `192.168.20.0/24` | `192.168.20.1` |
| Faro LAN | `192.168.30.0/24` | `192.168.30.1` |
| VMware WAN transport | `192.168.136.0/24` | Simulated transport network |

The pfSense WAN interfaces use stable laboratory addresses:

- Lisbon: `192.168.136.10`
- Porto: `192.168.136.20`
- Faro: `192.168.136.30`

The `192.168.136.0/24` network is not an internal TechSolutions LAN. It represents the simulated WAN transport used between the VPN peers.

### Centralized Services

| Service | Laboratory Implementation |
|---|---|
| Active Directory | `techsolutions.local` on `SRV-LISBOA` |
| DNS | Internal DNS on `192.168.10.82` |
| DHCP | Centralized Windows DHCP service |
| IIS | `http://portal.techsolutions.local` |
| MariaDB | Database service on TCP `3306` |
| SMB | `\\192.168.10.82\PartilhaTechSolutions` |

The laboratory was designed to validate dependencies and service behavior rather than reproduce every physical device from the enterprise reference architecture.

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

## Troubleshooting

Troubleshooting was performed using a controlled, evidence-driven methodology:

**Known-good baseline → Symptom → Evidence → Hypothesis → Root cause → Correction → Retest**

Five failures were deliberately introduced to validate the diagnostic process. An additional Active Directory secure channel failure was discovered naturally during final validation and investigated as a real incident.

| Incident | Primary Symptom | Root Cause | Final Validation |
|---|---|---|---|
| [01 — IPsec PSK Authentication Failure](incidents/01-ipsec-psk-authentication-failure.md) | Phase 1 authentication failed | Mismatched pre-shared key | Phase 1 `Established`, Phase 2 `Installed`, 0% packet loss |
| [02 — Phase 2 Selector Mismatch](incidents/02-phase2-selector-mismatch.md) | Phase 1 remained established but protected traffic failed | Incorrect remote network selector | Phase 2 `Installed`, 0% packet loss |
| [03 — Incomplete Firewall Policy](incidents/03-firewall-policy.md) | ICMP, DNS and SMB unavailable through an active VPN | Insufficient service-specific firewall permissions | ICMP, DNS and SMB independently restored |
| [04 — Incorrect DNS Configuration](incidents/04-dns-misconfiguration.md) | Internal hostname returned NXDOMAIN | Client using public DNS `8.8.8.8` instead of corporate DNS | `portal.techsolutions.local` resolved to `192.168.10.82` |
| [05 — Incorrect Default Gateway](incidents/05-default-gateway.md) | Remote network unreachable from `CLIENTE-PORTO` | Default gateway changed to `192.168.20.254` | Gateway restored to `192.168.20.1`, 0% packet loss |
| [06 — Active Directory Secure Channel Failure](incidents/06-ad-secure-channel.md) | `Test-ComputerSecureChannel` returned `False` | Required AD flows blocked by firewall policy | Dedicated least-privilege AD rules, repair successful, secure channel remained `True` |

### Diagnostic Principles

The incidents were used to separate failures across different layers rather than treating every connectivity problem as a VPN fault.

Key distinctions included:

- peer reachability vs. IKE authentication;
- Phase 1 state vs. Phase 2 traffic selectors;
- tunnel establishment vs. firewall authorization;
- IP connectivity vs. DNS resolution;
- infrastructure state vs. endpoint routing;
- ICMP reachability vs. Active Directory service dependencies.

Each incident document includes the observed symptom, evidence, diagnostic reasoning, root cause, correction and independent validation.

### Real Troubleshooting Case

The Active Directory secure channel failure was not deliberately introduced.

During final validation, `CLIENTE-PORTO` still showed domain membership but `Test-ComputerSecureChannel` returned `False`. ICMP connectivity to `SRV-LISBOA` remained operational while LDAP, Kerberos and RPC traffic was blocked.

A temporary diagnostic firewall permission confirmed the firewall as the failure point. It was then replaced with dedicated Active Directory TCP/UDP rules and aliases, the secure channel was repaired, the temporary permission was removed, and the final validation remained successful.

[Read the complete Active Directory incident →](incidents/06-ad-secure-channel.md)

## Project Status

**Completed laboratory implementation and validation.**

The final environment successfully demonstrated multi-site connectivity and remote access to centralized **Active Directory, DNS, IIS, MariaDB and SMB services**.

---

**Note:** This project was developed in an authorized laboratory environment for training and portfolio purposes. Credentials and other sensitive configuration values are intentionally excluded from the public documentation.
