# ADR-0008: Boot device and SSD tier layout

- **Status:** Accepted (provisional, review in the storage module)
- **Date:** 2026-10-23
- **Module:** 0.1 / host OS install
- **Supersedes:** none

## Context

All six data disks are in use: three SSDs for `ssd-vg`, three HDDs for `hdd-vg`.
No separate boot device is installed (no rear flex bays, IDSDM or internal USB
in use). RHEL needs an EFI System Partition (ESP) and `/boot` somewhere, and the
root filesystem must survive a single disk failure.

## Decision

Boot and root live on the SSD tier, built in the RHEL 9.5 installer:

| Mount | Device | Members | Metadata | Size |
| --- | --- | --- | --- | --- |
| `/boot/efi` | `md125` RAID1, FAT (ESP) | the two Samsung 850 PROs (partition 3) | 1.0 (at end of partition, so firmware reads each member as plain FAT) | 600 MiB |
| `/boot` | `md126` RAID1, xfs | the two 850 PROs (partition 2) | 1.2 | 1 GiB |
| PV for `ssd-vg` | `md127` RAID5, 512 KiB chunk | all three SSDs (partition 1) | 1.2 | 1.8 TiB |
| `/` | `ssd-vg/root`, thick LV, xfs | — | — | 50 GiB |
| swap | `ssd-vg/swap` | — | — | 8 GiB |
| (free) | rest of `ssd-vg` | — | — | reserved for the VM thin pool |

The firmware has two RHEL boot entries (`Boot0007`, `Boot0008`), one per ESP copy,
both in `BootOrder`.

HDDs (bays 3–5) were left untouched; `hdd-vg` is built in the storage module.

## Deviation from plan

The plan mirrored `/boot` and the ESP across **three** SSDs. The installer gave
the whole Samsung 870 EVO to the RAID5 member, so the boot mirrors have **two**
members, both 850 PROs. It still survives one disk failure. One boot copy sits on
the suspect bay 2 drive (S3D2NX0J603337T). Fixing it needs a rebuild; accepted.

## Alternatives considered

| Option | Why not |
| --- | --- |
| Separate boot device (BOSS, IDSDM, USB) | Hardware not installed; IDSDM/USB would be single devices |
| Root on the thin pool | A full thin pool could take down the host OS |
| Root as a plain partition | No redundancy |

## Consequences

- Root is a **thick** LV so thin-pool exhaustion can never take the OS down.
- Root shares the RAID5 with the VMs, including the suspect bay 2 drive; if it
  fails the array runs degraded and keeps working.
- The md device names (`md125–127`) differ from the charter's `/dev/md0`/`md1`.
  The storage module should name arrays by `--name` and reference them by UUID,
  and correct the charter.
- Firmware writes to an ESP member can desync the RAID1 mirror; resync after any
  firmware-driven ESP change.
