# Troubleshooting Log (AD DS / DNS / DHCP / Domain Join)

This page summarises the real issues I hit while building the lab and how I fixed them. It’s written as “symptom → cause → fix → verification” so it reads well in internal.

---

## 1) AD DNS zones/records not populating (`_msdcs` missing)
Symptom  
- After promoting the DC, expected forward lookup zones/AD DNS records weren’t present.
- Group Policy / AD functionality was effectively broken because DC discovery couldn’t work.

Likely cause  
- AD DNS infrastructure wasn’t being created correctly (missing `_msdcs` implies DC/service discovery failure).
- Initial DC/DNS configuration issues during promotion.

Fix  
- Checked NIC DNS settings (ensured the DC points to itself as DNS).
- Demoted and re-promoted the DC to rebuild AD/DNS cleanly.
- When the issue persisted under `mirai.example`, rebuilt the domain using an internal-only suffix (`.local`) to avoid “internet-looking” domain behaviour in a lab.

Verification  
- `_msdcs` zone appears and contains SRV records for LDAP/Kerberos.
- Clients can perform SRV lookups and discover DC services.

What I learned  
- AD is tightly coupled to DNS SRV records; if service discovery fails, Kerberos/LDAP/GPO break.

---

## 2) DC demotion problems during rebuild
Symptom  
- Trouble demoting the DC during rework.

Likely cause  
- Incomplete DNS state (reverse lookup zone absence/poor DNS health made the DC identity and related checks unstable).

Fix  
- Force demoted the DC, corrected DNS approach, then re-promoted using a `.local` domain and rebuilt reverse lookup properly.

Verification  
- Domain rebuild completed; reverse lookup zone present and PTR record creation works.

---

## 3) DNS resolution failure due to incorrect DNS server IP on the DC NIC
Symptom  
- `nslookup` / name resolution failed after setting DNS server to an incorrect value (entered a 169.x.x.x-style address).

Cause  
- Wrong DNS server configured on the DC’s network settings.

Fix  
- Corrected DC NIC DNS setting so the server resolves via the intended DNS configuration (self for AD DNS).

Verification  
- `nslookup` begins returning expected results after correcting the address.

---

## 4) Client couldn’t join the domain (DNS + network dependency)
Symptom  
- Domain join failed even with correct credentials.

Cause  
- Client wasn’t using the DC as DNS (manual DNS wasn’t set yet / DHCP not supplying correct option).
- Network mode (bridged) meant the home router was doing DHCP/DNS instead of the lab, so the client wasn’t discovering the DC correctly.

Fix  
- Switched the lab to a private virtual network (host-only / internal approach) so the lab controlled DHCP/DNS.
- Ensured the client DNS points to DC01 (either via DHCP options or manual DNS during testing).
- Activated the DHCP scope (so clients get correct DNS automatically).

Verification  
- Client can resolve the domain/DC.
- Domain join succeeds; reboot; domain logon works.

What I learned  
- Domain join failures are often network/DNS, not credentials.

---

## 5) DHCP scope design mismatch (wrong subnet / L2-only limitation)
Symptom  
- DHCP scope wouldn’t work as expected / clients weren’t receiving usable addresses for DC communication.

Cause  
- Scope range didn’t match the subnet the DC/clients were actually operating in.
- In an L2-only lab, clients must be on the same subnet unless routing is introduced.

Fix  
- Corrected the scope to a /24 matching the lab subnet.
- Revisited scope options (DNS + default gateway) to match lab design.

Verification  
- Client receives an address in the expected range.
- Client can ping/resolve DC and proceed with domain join.

---

## 6) Rogue DHCP (unexpected DHCP source)
Symptom  
- Clients received “wrong” network settings (unexpected leases), causing domain join and DNS issues.

Cause  
- Hypervisor networking defaults can provide DHCP (or route to an upstream DHCP) depending on NAT/host-only configuration.

Fix  
- Standardised the lab network mode to ensure only the intended DHCP server is active.
- Disabled/removed the unwanted DHCP source (hypervisor-side) and renewed leases.

Verification  
- `ipconfig /all` shows DHCP server = DC01.
- DHCP lease table on DC shows the client lease.

---

## 7) Cloned domain-joined VM broke trust with the domain
Symptom  
- A cloned client could no longer communicate properly with AD / trust relationship issues.

Cause  
- Domain-joined machines have a machine account password/secure channel (Netlogon). Cloning duplicates identity state and breaks trust.

Fix  
- Removed the computer from the domain and re-joined cleanly (or rebuild the client).
- Future prevention: `sysprep /generalize /oobe /shutdown` before cloning, or remove from domain first.

Verification  
- Re-joined client authenticates successfully and can log on to the domain.

---

## 8) Management tooling friction on the DC (browser security zone)
Symptom  
- Internet browsing blocked/restricted on the DC (IE security zone too strict), impacting downloading/management setup steps.

Cause  
- Default server hardening / security zone restrictions.

Fix  
- Moved management tooling to a separate management host (admin centre/RSAT mindset) instead of relying on browsing directly on the DC.

Verification  
- Admin tasks performed from the management host without weakening DC posture.

---
