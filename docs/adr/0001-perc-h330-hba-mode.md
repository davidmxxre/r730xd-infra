# ADR-0001: PERC H330 Mini in HBA mode under mdadm

- **Status:** Accepted
- **Date:** 2026-10-23
- **Module:** 0.1 Hardware preparation and out-of-band management
- **Supersedes:** none

## Context

The R730xd's integrated storage controller is a **PERC H330 Mini** (LSI SAS3008,
firmware 25.5.9.0001). The project charter and roadmap say "PERC H730"; that
was wrong. The controller was identified on 2026-10-23 three ways:

- `lspci -nn` shows `MegaRAID SAS-3 3008 [Fury]` (an H730 shows `3108`).
- A stale UEFI boot entry named the controller "PERC H330 Mini".
- `racadm swinventory` lists `PERC H330 Mini`.

The H330 has no cache memory and no battery. Battery events from May 2026 in the
Lifecycle Controller log (`BAT1029`, `BAT1008`) came from an earlier controller
and are historical.

The platform design puts Linux software RAID (`mdadm`) and LVM on the disks, so
the OS must see real disks, not controller-built virtual disks.

## Decision

Run the H330 in **HBA mode** (personality switch). Every disk is passed through
to the OS and reported by iDRAC as `State = Non-Raid`. No virtual disks exist.

The controller was already in HBA mode when the module started
(`CurrentControllerMode = HBA`); no switch was needed.

## Alternatives considered

| Option | Why not |
| --- | --- |
| RAID mode, one RAID0 virtual disk per drive | Controller metadata on every disk, SMART hidden behind the controller, disk swaps need controller reconfiguration. Double-stacked RAID logic |
| Hardware RAID5 on the H330, no mdadm | The H330 has no cache, so parity RAID performs poorly; arrays are tied to a compatible PERC; defeats the project's purpose of operating Linux storage |
| RAID mode with non-RAID disks | Works, but keeps a RAID personality that will never be used |

## Consequences

- mdadm sees raw disks. Arrays are portable to any HBA if the H330 dies.
- **The H330 is a single point of failure:** all six front disks hang off it.
  HBA mode makes that failure recoverable (arrays import on a replacement), not
  avoidable.
- **The RAID5 write hole belongs to mdadm.** The storage module must choose a
  mitigation (mdadm PPL or a journal device) and record it in an ADR.
- **SMART must be monitored from the OS** (`smartd`). iDRAC's drive-health view
  in HBA mode is not relied on.
- **Locate LEDs do not work in HBA mode on this server.** `racadm storage blink`
  returns success but the LED stays solid green. Bays are identified by serial
  number and the bay map, or by pulling a drive while the disks are empty.
- `sdX` names are not stable across boots. Everything persistent uses
  `/dev/disk/by-id/` paths.
