# ADR-0003: iDRAC authentication — one named local admin, SSH keys, root disabled

- **Status:** Accepted
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** the original 3-person design (three named admins)

## Context

Every R730 ships with `root`/`calvin` in user slot 2. The project is now solo.
FreeIPA will run on this host in Phase 1, and the iDRAC is needed most when the
host is down — exactly when LDAP would be unreachable.

## Decision

| Slot | User | State |
| --- | --- | --- |
| 1 | (reserved) | — |
| 2 | `root` | **Disabled**, password scrambled to a random 20-character value first |
| 3 | `dave` | Enabled, `Privilege=0x1ff` (Administrator), IPMI LAN/serial privilege 15 (No Access), SSH key in key slot 1 |
| 4–16 | empty | Disabled. Slot 6 reserved for a read-only metrics account in Phase 1 |

- Daily access is SSH key authentication with a **dedicated RSA 4096 key**
  (`~/.ssh/id_rsa_idrac`). iDRAC8's key store predates Ed25519.
- The `dave` password (maximum 20 characters on iDRAC8) is used for the web UI,
  Virtual Console and remote RACADM.
- **No LDAP/FreeIPA integration** for the iDRAC.

## Alternatives considered

| Option | Why not |
| --- | --- |
| Keep root with a strong password | Known account name; no attribution of actions |
| FreeIPA/LDAP login | FreeIPA runs on this host; the iDRAC must work when the host is down |
| Several named accounts | Solo project; one admin is the honest model |

## Consequences

- **Lockout recovery:** physical console → F2 at POST → iDRAC Settings → User
  Configuration. Because the server is on site, lockout is recoverable.
- Losing the private key does not lock you out; the password still works for
  the web UI and remote RACADM.
- iDRAC IP blocking (5 failures in 60 s → 10-minute block, ADR-0004) also applies
  to the admin. Slow down after two failed logins.
- If a second person ever joins, add a named slot for them; never share `dave`.
