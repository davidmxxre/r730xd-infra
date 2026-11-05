# ADR-0005: Firmware baseline and freeze

- **Status:** Accepted
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** none

## Context

The R730xd and iDRAC8 are end-of-life; Dell has stopped publishing most updates
for this platform. Older firmware has known vulnerabilities, and later update
windows may not exist as Dell withdraws downloads.

## Decision

1. Bring every component to its latest available release once, then **freeze**.
2. The check on 2026-10-23 found every component already current, or with
   nothing newer published. **No firmware was flashed in Module 0.1.**

   | Component | Version | Basis |
   | --- | --- | --- |
   | iDRAC8 with Lifecycle Controller | 2.86.86.86 | Final iDRAC8 release |
   | BIOS | 2.19.0 | Latest R730/R730xd BIOS |
   | PERC H330 Mini | 25.5.9.0001 | Newest seen |
   | Backplane BP13G+EXP | 3.35 | Latest 13G expander firmware |
   | rNDC Intel X520/I350 | 22.5.7 | Current |
   | System CPLD | 1.0.1 | No CPLD package listed on Dell support |
   | PSUs (Liteon 750 W, Dell PN W8R3C) ×2 | 00.23.32 | No package listed for this part number |
3. **Method for any future update:** download from Dell support for service tag
   5VWPHK2, verify the SHA-256, push with remote RACADM
   (`racadm -r <ip> -u dave -p … update -f <file>`) or the web UI
   (iDRAC Settings → Update and Rollback). No Lifecycle Controller online
   updates: they would need an internet route from the management network.
4. **Ordering rules:** iDRAC first; BIOS next; then PERC, NIC, others. **A CPLD
   update is applied alone**, never batched. Never flash a package built for a
   different part number (PSU packages are per part number).

## Consequences

- Nothing to archive from this module, since nothing was downloaded. **Action:**
  archive copies of the current packages (at least iDRAC 2.86.86.86, BIOS 2.19.0,
  H330 25.5.9.0001) off-host while Dell still serves them, so a replacement
  board can be brought to this baseline.
- Firmware changes after this point go through an RFC once Phase 4 exists.
