# Incident 06 — Active Directory Secure Channel Failure

## Summary

During final validation, an additional problem was identified that was not part of the five planned troubleshooting scenarios.

`CLIENTE-PORTO` still appeared to belong to the `techsolutions.local` domain, but its Active Directory secure channel was broken.

Basic IP connectivity to `SRV-LISBOA` remained available, while several service-specific Active Directory ports were blocked by the firewall policy.

This incident became a real troubleshooting case within the laboratory and demonstrated that successful VPN connectivity does not guarantee functional domain communication.

## Environment

| Component | Configuration |
|---|---|
| Client | `CLIENTE-PORTO` |
| Domain | `techsolutions.local` |
| Domain controller / server | `SRV-LISBOA` |
| Server IP | `192.168.10.82` |
| LDAP test | TCP `389` |
| Kerberos test | TCP `88` |
| RPC Endpoint Mapper test | TCP `135` |
| Validation command | `Test-ComputerSecureChannel` |

## Initial Condition

During final verification, `CLIENTE-PORTO` continued to show local domain membership, but domain trust functionality was not operating correctly.

This discrepancy required validation of the machine secure channel rather than relying only on the local indication that the computer belonged to the domain.

## Initial Symptom

The secure channel was tested with:

`Test-ComputerSecureChannel -Verbose`

<p align="center">
  <img src="../assets/troubleshooting/ad-secure-channel-broken.png"
       alt="Broken Active Directory secure channel between CLIENTE-PORTO and techsolutions.local"
       width="80%">
</p>

The command returned:

`False`

This confirmed that the trust relationship between `CLIENTE-PORTO` and the domain was not operational.

## Network Layer Investigation

Basic connectivity to `SRV-LISBOA` was still available through ICMP.

Because general IP reachability existed, the investigation moved to service-specific Active Directory dependencies.

The following TCP services were tested individually:

- LDAP — TCP `389`
- Kerberos — TCP `88`
- RPC Endpoint Mapper — TCP `135`

These services were initially blocked by the firewall policy.

## Diagnostic Hypothesis

The working hypothesis was that the VPN tunnel itself was functional, but the firewall did not permit all flows required by Active Directory.

To test this hypothesis, a temporary diagnostic rule was applied with traffic restricted to:

`CLIENTE-PORTO` → `SRV-LISBOA`

The purpose of this rule was diagnostic only.

If the same port tests began succeeding immediately after the temporary rule was applied, the firewall policy would be confirmed as the failure point.

## LDAP Validation

After the temporary firewall adjustment, LDAP became reachable.

<p align="center">
  <img src="../assets/troubleshooting/ad-ldap-389-reachable.png"
       alt="LDAP TCP port 389 reachable after temporary firewall adjustment"
       width="80%">
</p>

The test returned:

`TcpTestSucceeded : True`

This demonstrated that TCP `389` traffic could reach the domain controller when the firewall allowed it.

## Kerberos Validation

Kerberos connectivity was then tested.

<p align="center">
  <img src="../assets/troubleshooting/ad-kerberos-88-reachable.png"
       alt="Kerberos TCP port 88 reachable after temporary firewall adjustment"
       width="80%">
</p>

TCP port `88` became reachable.

This confirmed that the tested Kerberos flow had also been blocked by the previous firewall policy.

## RPC Validation

RPC Endpoint Mapper connectivity was tested next.

<p align="center">
  <img src="../assets/troubleshooting/ad-rpc-135-reachable.png"
       alt="RPC Endpoint Mapper TCP port 135 reachable after temporary firewall adjustment"
       width="80%">
</p>

TCP port `135` also became reachable.

The combined service results strongly supported the firewall hypothesis.

## Diagnostic Reasoning

| Observation | Interpretation |
|---|---|
| Client still showed domain membership | Local membership alone did not prove trust health |
| `Test-ComputerSecureChannel` returned `False` | Machine-domain secure channel was broken |
| ICMP to `SRV-LISBOA` worked | General IP reachability existed |
| LDAP 389 was blocked | Required AD traffic was unavailable |
| Kerberos 88 was blocked | Authentication-related flow was unavailable |
| RPC 135 was blocked | Another required domain dependency was unavailable |
| Temporary restricted rule made tests succeed | Firewall policy confirmed as root cause |

### What These Tests Proved

The tests proved that the VPN and basic IP path were operational, but required Active Directory flows were being blocked by the firewall policy.

### What These Tests Did Not Prove

Successful ICMP did not prove that domain services were functional.

Likewise, a single reachable AD port would not prove that the complete secure channel dependency set was available.

The trust relationship therefore had to be repaired and validated separately after the permanent policy was applied.

## Root Cause

The final IPsec firewall policy did not yet permit all traffic required for Active Directory communication between the Porto network and `SRV-LISBOA`.

The VPN tunnel was operational, but domain-specific service flows were blocked.

This prevented the client from maintaining a healthy secure channel with the domain.

## Permanent Correction

After confirming the firewall as the cause, dedicated Active Directory aliases and permanent TCP/UDP firewall rules were created.

The destination was restricted to `SRV-LISBOA`.

The temporary diagnostic permission was not retained as the final solution.

## Secure Channel Repair

The trust relationship was repaired using:

`Test-ComputerSecureChannel -Repair`

<p align="center">
  <img src="../assets/troubleshooting/ad-secure-channel-repair-success.png"
       alt="Active Directory secure channel successfully repaired"
       width="80%">
</p>

The repair returned:

`True`

This indicated that the secure channel had been successfully re-established.

## Final Least-Privilege Firewall Policy

The temporary broad diagnostic rule was removed after the cause had been confirmed.

The final firewall configuration used dedicated Active Directory rules and aliases.

<p align="center">
  <img src="../assets/troubleshooting/ad-final-least-privilege-firewall-policy.png"
       alt="Final least-privilege Active Directory firewall policy across the IPsec VPN"
       width="95%">
</p>

The permanent policy explicitly allowed the required Active Directory flows while keeping the destination limited to the centralized server.

## Final Secure Channel Validation

The secure channel was tested again after the temporary diagnostic rule had been removed.

<p align="center">
  <img src="../assets/troubleshooting/ad-secure-channel-final-validation.png"
       alt="Final Active Directory secure channel validation after applying permanent firewall rules"
       width="80%">
</p>

`Test-ComputerSecureChannel` returned:

`True`

and reported that the secure channel was in:

**good condition**

This final validation demonstrated that the permanent least-privilege firewall policy was sufficient and that the recovery did not depend on the temporary diagnostic rule.

## Incident Timeline

| Stage | Result |
|---|---|
| Discovery | Additional unplanned issue found during final validation |
| Initial secure channel test | `False` |
| Basic connectivity | ICMP to `SRV-LISBOA` operational |
| LDAP test | TCP 389 initially blocked |
| Kerberos test | TCP 88 initially blocked |
| RPC test | TCP 135 initially blocked |
| Diagnostic action | Temporary restricted firewall rule applied |
| Diagnostic result | AD port tests changed to successful |
| Root cause | Required Active Directory traffic blocked by firewall policy |
| Permanent correction | Dedicated AD TCP/UDP aliases and rules created |
| Repair | `Test-ComputerSecureChannel -Repair` returned `True` |
| Security cleanup | Temporary broad diagnostic rule removed |
| Final validation | Secure channel remained `True` and in good condition |

## Key Technical Takeaway

A working site-to-site VPN and successful ICMP reachability do not prove that Active Directory is operational across the tunnel.

Domain functionality depends on multiple service-specific flows.

Troubleshooting therefore needs to distinguish between:

- general IP reachability;
- VPN state;
- firewall authorization;
- LDAP connectivity;
- Kerberos connectivity;
- RPC connectivity;
- machine secure channel state.

The strongest evidence in this incident came from changing only the firewall condition, observing the AD port tests immediately succeed, replacing the diagnostic rule with dedicated least-privilege rules, repairing the trust relationship and then confirming that the secure channel remained healthy after the temporary rule was removed.
