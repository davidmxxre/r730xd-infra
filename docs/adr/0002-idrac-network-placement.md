# ADR-0002: iDRAC network placement

- **Status:** Accepted (provisional addressing)
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** none

## Context

The iDRAC is the only way to reach the server's console, power and BIOS when the
host OS is down. It needs a network path that does not depend on anything
running on the host.

The OPNsense edge firewall will run **as a VM on this same R730xd**. The
management VLAN therefore cannot exist until the host OS and OPNsense are built.
The project is run by one person (Dave), who is physically at the server.

## Decision

1. Use the **dedicated iDRAC port**, not a shared LOM, with **static IPv4**.
   IPv6 off. Dynamic DNS registration off.
2. **Provisional addressing until the network module:** a direct cable from the
   admin workstation to the iDRAC port on an isolated link, `192.168.0.0/24`.

   | Device | Address |
   | --- | --- |
   | iDRAC | 192.168.0.120/24, gateway 192.168.0.1 (unused) |
   | Admin VM `app1` (bridged NIC) | 192.168.0.10/24, no gateway |
   | Mac USB Ethernet adapter (when needed) | 192.168.0.99/24, no router |

   The link never touches the home LAN (192.168.1.0/24).
3. In the network module, move the iDRAC to an untagged access port on the
   management VLAN (`racadm set iDRAC.IPv4.Address …`) and record it here.
4. Every iDRAC dependency is configured **by IP**, never by an internal DNS name.
   Internal DNS will run on this host and must not be required to reach it.
5. **Time source:** provisional NTP server is the admin VM (`chronyd` with
   `allow 192.168.0.120`). Moves to OPNsense in the network module. A VM time
   source is acceptable for the iDRAC because it keeps its own clock; only DNS
   and authentication must never depend on the host.

## Alternatives considered

| Option | Why not |
| --- | --- |
| Shared LOM | Couples out-of-band access to host NIC state and bonding |
| VLAN tag on the iDRAC | Adds a dependency on trunk configuration for no gain |
| DHCP | Makes the iDRAC depend on a DHCP server |
| OPNsense on separate hardware | Better resilience, but no spare hardware; see Consequences |

## Consequences

- **Accepted single point of failure:** with OPNsense on this host, remote
  out-of-band access disappears whenever the host is down. Impact is **low**
  while Dave is the only operator and is on site. **Revisit before relocating**,
  when remote out-of-band access becomes the primary access path.
- Until the network module, out-of-band access requires the workstation to be
  physically cabled to the iDRAC port.
- When the admin VM is off, the iDRAC has no NTP source and drifts slowly.
