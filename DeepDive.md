This page captures the “concept learning” parts of the project that I wanted to know more about: Wireshark onboarding/authentication flow, DHCP behaviour, and AD DNS service discovery.

---

## 1) Wireshark: DC onboarding + authentication (what I captured and why)

### Goal
Understand what actually happens during:
1) Domain join (pre-reboot)
2) First domain logon (post-reboot)

### High-level flow I validated
- Domain join: DNS discovery → SMB session (admin join) → LDAP directory operations → RPC/Netlogon secure channel → reboot
- Post-reboot logon: Kerberos tickets → LDAP queries → RPC (LSA/SAMR context) → SMB access to NETLOGON/SYSVOL (policy/scripts)

### Protocols I observed (what they mean)
- ARP: IP→MAC resolution on the local subnet (basic readiness)
- DNS: DC/service discovery via SRV records (e.g., `_ldap._tcp.dc._msdcs...`)
- SMB2 (445): authenticated sessions to IPC$, NETLOGON/SYSVOL access
- NTLMSSP: seen during initial join phases (before Kerberos is fully established)
- LDAP (389/636): directory binds/lookups/object creation
- DCE/RPC (135 + dynamic ports): endpoint mapper + Netlogon/LSA/SAMR functions
- Netlogon RPC: machine secure channel creation/maintenance
- Kerberos (88): TGT + service tickets for LDAP/SMB

### Example display filters (practical)
- `arp`
- `dns`
- `kerberos`
- `ldap`
- `ntlmssp`
- `smb2`
- `tcp.port == 445`
- `tcp.port == 389 || tcp.port == 636`
- `tcp.port == 88 || udp.port == 88`
- `tcp.port == 135`

### What this taught me (short)
- AD “working” depends on DNS SRV discovery first.
- Join often starts with NTLM in SMB/LDAP, then stabilises into Kerberos-based access post-reboot.
- A lot of “Windows magic” is actually repeatable, observable protocol steps.

---

## 2) DHCP: what I did + what I learned

### What I configured
- DHCP role on the DC.
- A scope aligned to the actual lab subnet.
- Options set for:
  - DNS server = DC01 (critical for AD)
  - Default gateway = firewall (egress/logging only)

### Key lessons
- If scope/subnet don’t match, clients get unusable config (or never get leases).
- In L2-only lab segments, clients must be on the same subnet as the DC unless routing is introduced.
- Rogue DHCP is common in virtual labs; one wrong network mode and clients pull leases from the wrong place.

### How I validated DHCP was correct
- On client: `ipconfig /all` shows DHCP server + DNS server as expected.
- On DC: DHCP console shows active leases.

---

## 3) DNS deep dive: why AD DNS is different

### Record types I used (and why)
Host discovery
- `A` / `AAAA`: hostname → IP
- `PTR`: IP → hostname (reverse lookup)

Service discovery
- `SRV`: tells clients where a service lives (host + port), e.g. LDAP/Kerberos DC discovery

Zone authority/integrity
- `NS`: who is authoritative for the zone
- `SOA`: primary zone metadata + replication behaviour
- `CNAME`: alias to another hostname

### `_msdcs` (why it matters)
- `_msdcs` is AD-specific DNS infrastructure that helps clients locate DCs and services in the forest.
- When it was missing/broken, DC discovery failed and the downstream impact was Kerberos/LDAP/GPO failures.

### What I want to be able to explain clearly
- Why SRV records are required for AD (clients don’t “guess” where LDAP/Kerberos live).
- Why “client DNS must point to the DC” is non-negotiable in most single-DC labs.
- The difference between forward lookup and reverse lookup in troubleshooting.

---

## Optional next deep dives (later)
- Add a second DC and observe replication-related DNS and AD behaviour.
- Enable auditing + correlate security events with observed Kerberos/LDAP flows.
- Centralise logs (WEF) and build a simple “identity security monitoring” story.
