# ADR-0004: iDRAC protocol surface

- **Status:** Accepted
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** none

## Context

iDRAC8 reached end of software maintenance in February 2024; **2.86.86.86 is the
final firmware**. No more security fixes will ship. The only way left to reduce
risk is to switch off what isn't needed and limit who can reach it. A compromised
iDRAC means a compromised host: power, console, virtual media and BIOS.

## Decision

| Service | Port | State | Reason |
| --- | --- | --- | --- |
| SSH | TCP 22 | **On**, key auth | Primary admin interface; serial console via `console com2` |
| HTTPS (web UI, remote RACADM, Redfish) | TCP 443 | **On, TLS 1.2 only** (`iDRAC.WebServer.TLSProtocol=2`) | Needed for the web UI, Virtual Console and file uploads. TLS 1.0/1.1 deprecated |
| HTTP | TCP 80 | On | Redirects to HTTPS only |
| Virtual Console (HTML5) | TCP 5900 | **On, encrypted** (`EncryptEnable=Enabled`) | Remote screen for POST/F10/F2 work without a monitor |
| VNC server | TCP 5901 | **Off** | Unencrypted console path |
| Telnet | TCP 23 | **Off** | Cleartext |
| IPMI over LAN | UDP 623 | **Off** (`iDRAC.IPMILan.Enable=Disabled`) | IPMI 2.0's RAKP login leaks crackable password hashes to unauthenticated clients |
| SNMP agent | UDP 161 | **Off** (`iDRAC.SNMP.AgentEnable=Disabled`) | Was on with community `public`. Off until Phase 1 picks SNMPv3 or Redfish for metrics |
| IP blocking | — | **On**: 5 failures / 60 s → 600 s block | Brute-force protection |

## Accepted residuals

- **UDP 623 answers RMCP presence pings** even with IPMI disabled, so nmap reports
  it `open`. An IPMI capabilities request (`nmap --script ipmi-version`) gets no
  answer, and `ipmitool` cannot open a session. No setting closes the port;
  harmless.
- `iDRAC.IPMILan.EncryptionKey` (the KG key) is still all zeros. Irrelevant while
  IPMI is off; set a random key before ever enabling IPMI.
- `iDRAC.IPMILan.CommunityName` and `iDRAC.SNMP.AgentCommunity` are still
  `public`. Both services are off. Change them if either is ever enabled.
- No BIOS setup password (`PasswordStatus=Unlocked`): anyone at the physical
  console can change BIOS settings. Acceptable in a home location.
- `VirtualConsole.MaxSessions=6`; could be lowered for a solo setup. Not changed.

## Verification (from another machine on the link)

```
nmap -Pn -sT -p 22,23,80,443,5900,5901 192.168.0.120     # 22,80,443,5900 open; 23,5901 closed
nmap -Pn -p 443 --script ssl-enum-ciphers 192.168.0.120  # TLSv1.2 only, all ciphers grade A
sudo nmap -Pn -sU -p 161,623 192.168.0.120               # 161 closed; 623 open (RMCP)
sudo nmap -Pn -sU -p 623 --script ipmi-version 192.168.0.120  # no ipmi-version section
ipmitool -I lanplus -H 192.168.0.120 -U x -P x chassis status  # "Unable to establish IPMI v2 / RMCP+ session"
```

## Consequences

- No SSH crypto workaround was needed: RHEL 9's DEFAULT policy negotiates with
  this iDRAC.
- Phase 1 monitoring must choose a metrics path (SNMPv3 with a non-default user,
  or Redfish over HTTPS with a read-only account) and amend this ADR.
- Phase 6 range isolation must prove TCP 22/443/5900 and UDP 623 to the iDRAC are
  unreachable from the range.
