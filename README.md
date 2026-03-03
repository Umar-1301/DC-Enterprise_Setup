# Active Directory Lab (Windows Server 2019) — AD DS + DNS/DHCP, Domain Join, and Troubleshooting Walkthrough

This repo is a single README walkthrough of an Active Directory lab I built to strengthen enterprise identity fundamentals and demonstrate real troubleshooting ability (DNS, DHCP, subnetting, domain join, and machine trust).

Note on the firewall: A Palo Alto firewall exists only as the default gateway for outbound internet access and visibility/logging. Firewall configuration is out of scope here.

## What I built (current state)
- 1x Domain Controller (Windows Server 2019): AD DS + DNS + DHCP
- 1x Member server (Windows Server): management host (admin tooling)
- 1x Windows client: domain-joined endpoint
- Lab networking in a hypervisor environment (VirtualBox), including changes between network modes during troubleshooting

## High-level topology
- Clients and servers are on the same lab subnet once stable.
- The DC provides DNS to domain members.
- DHCP issues addresses and sets DNS/gateway options.
- Default route points to the firewall for outbound egress and logging.

- DC01: static IP
- Member Server: DHCP
- Client: DHCP / Static IP for testing
- DNS: DC01
- Default gateway: firewall

---

# Walkthrough (what I did)

## 1) Built the Domain Controller
- Installed Windows Server 2019 in a VM and applied updates.
- Set a static IP on the DC (to keep DNS/AD stable).
- Installed AD DS and promoted the server to a Domain Controller.
- Installed/verified DNS as part of AD DS.

Outcome: I had a working domain and a DC capable of servicing DNS queries for domain members.

## 2) Fixed AD DNS health (service discovery / AD zones)
During setup I ran into DNS issues that showed why AD DNS is “not just normal DNS”.
- AD relies on DNS records (especially SRV records) for DC discovery (LDAP/Kerberos).
- Missing/incorrect AD DNS data breaks domain join, Kerberos, GPO processing, and directory operations.
- I resolved this by correcting DNS/DC configuration and re-running promotion steps cleanly (demote/re-promote approach when required).

Outcome: DNS became reliable for DC discovery and domain operations.

## 3) Installed and configured DHCP
- Installed the DHCP role on the DC.
- Created a DHCP scope and activated it.
- Set essential scope options:
  - DNS server = DC01
  - Default gateway = firewall (egress/logging)

Troubleshooting I encountered and fixed:
- Scope/subnet mismatch (clients weren’t in the subnet the scope was serving).
- “Rogue DHCP” behaviour introduced by hypervisor defaults/network mode changes (clients pulling incorrect leases).

Outcome: Clients consistently received correct IP settings and could resolve the domain via the DC.

## 4) Joined machines to the domain (member server + client)
I built and domain-joined:
- A member server (used as a management host).
- A Windows client endpoint.

Key lesson: domain join success depends heavily on:
- Subnet reachability (L3 routing if not on same subnet)
- Correct DNS (DC as DNS server for clients)

When domain join failed, the root cause was not credentials — it was network/DNS dependencies.

Outcome: Both systems joined successfully and domain logon worked.

## 5) Organised AD (basic structure + groups)
- Created a basic OU structure (enough to separate users/workstations/servers for clean administration and future GPO targeting).
- Created security groups and documented group scope/type decisions.

Outcome: The domain was no longer “flat”; it had structure that can scale into policy targeting and least privilege.

## 6) Touched on security controls (what I’ve implemented so far)
These are the security-relevant areas I worked on in this lab:

- Least privilege mindset:
  - Standard users vs admin usage (avoid day-to-day admin where possible)
- Password policy thinking:
  - Fine-grained password policy concepts and account lockout trade-offs (lockouts can reduce spraying, but can also be abused for DoS)
- Secure administration approach:
  - Used a management host approach (Windows Admin Center / RSAT mindset) to reduce the need to RDP into the DC

Outcome: I can explain why these controls matter, how they work, and where they fit in an enterprise identity model.

## 7) Validated the protocol flow (what happens “on the wire”)
To properly understand domain join and logon, I validated traffic patterns at a protocol level:
- ARP and basic L2 resolution
- DNS queries (including SRV-based DC discovery)
- SMB/NTLM phases during join steps
- LDAP operations for directory actions
- RPC / endpoint mapper and Netlogon secure channel
- Kerberos authentication flow during normal domain usage

Outcome: I can describe domain join/logon as a real sequence of dependent services (not “magic AD”).

## 8) Learned why cloning breaks domain trust (and how to fix it)
In virtual labs it’s common to clone VMs. I validated why this breaks:
- Domain-joined machines have a unique machine account relationship with the domain.
- Cloning without sysprep/generalize can duplicate identity/trust state.
- Fixes include sysprep before cloning or re-joining the domain cleanly.

Outcome: I can explain machine trust failures and remediation confidently.

---

# What this project demonstrates (skills)
- Windows Server build and core infra setup (AD DS / DNS / DHCP)
- Real troubleshooting: subnetting, DNS service discovery, DHCP scope integrity, hypervisor networking pitfalls
- Practical identity fundamentals (domain join, machine trust, authentication flows)
- Security awareness (least privilege, password/lockout thinking, secure admin patterns)

---

# Further improvements (next small upgrades)
These are fast, high-impact additions that elevate the project without expanding scope too much:

1) Enable Advanced Audit Policy via GPO and document 2–3 example Event IDs
- Successful logon, failed logon, user/group change

2) Implement Windows LAPS
- Rotate local admin passwords on endpoints and document how retrieval works

3) Minimal baseline GPO set (small but meaningful)
- Workstation firewall baseline
- Restrict local admin membership
- Disable legacy/weak settings where appropriate

4) Add a second DC (resilience) and document why it matters
- Introduce replication concepts and failure tolerance

5) Centralise logs (lightweight)
- Windows Event Forwarding (WEF) to a collector or a simple SIEM target later

---
