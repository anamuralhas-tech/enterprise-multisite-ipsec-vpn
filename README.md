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

## Project Status

**Completed laboratory implementation and validation.**

The final environment successfully demonstrated multi-site connectivity and remote access to centralized **Active Directory, DNS, IIS, MariaDB and SMB services**.

---

> **Note:** This project was developed in an authorized laboratory environment for training and portfolio purposes. Credentials and other sensitive configuration values are intentionally excluded from the public documentation.
