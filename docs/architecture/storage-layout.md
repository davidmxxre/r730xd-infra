# Storage layout: r730xd-01

As built, 2026-11-09 (Modules 0.1 and 0.2). Kernel device letters (`sdX`)
and md numbers are from that day and can change at any boot. Serial
numbers, array names and UUIDs do not.

## Stack

```
 Bay  Disk (serial)                    Partitions            Array (name)            LVM                       Mount
 ───  ───────────────────────────────  ────────────────────  ──────────────────────  ────────────────────────  ─────────────────────
  0   Samsung 870 EVO  S75BNL0X441461N  p1 931.5G ─────────┐
  1   Samsung 850 PRO  S3D2NX0J603541M  p1 931.5G ─────────┼─ pv00  RAID5 1.8T ───── ssd-vg ─┬ root    50G     /
  2   Samsung 850 PRO  S3D2NX0J603337T  p1 931.5G ─────────┘   512K chunk, bitmap            ├ swap     8G     [swap]
                                                                                             ├ vmpool  ~1.6T   (thin pool, VM disks)
                                                                                             └ free    ~180G   (root snapshots)
  1   850 PRO ...541M                   p2 1G ─────┐
  2   850 PRO ...337T                   p2 1G ─────┴────────── boot  RAID1 1023M ──────────────────────────── /boot
  1   850 PRO ...541M                   p3 601M ───┐
  2   850 PRO ...337T                   p3 601M ───┴────────── boot_efi RAID1 600.9M ──────────────────────── /boot/efi

  3   Seagate ST1000DM010  W9A63479     whole disk ────────┐
  4   Seagate ST1000DM010  W9A7NB9T     whole disk ────────┼─ hdd   RAID5 1.8T ───── hdd-vg ─┬ backups  1T     /srv/backups
  5   Seagate ST1000DM010  W9A9ZDXL     whole disk ────────┘   512K chunk, PPL               ├ bulk   300G     /srv/bulk
                                                                                             ├ isos   150G     /var/lib/libvirt/isos
                                                                                             └ free  ~388G
```

## Physical disks

Bays are numbered down each column, left to right.

| Bay | Model | Serial | Stable path | Role |
|---|---|---|---|---|
| 0 | Samsung SSD 870 EVO 1TB | S75BNL0X441461N | `/dev/disk/by-id/ata-Samsung_SSD_870_EVO_1TB_S75BNL0X441461N` | `pv00` member. One partition spanning the disk |
| 1 | Samsung SSD 850 PRO 1TB | S3D2NX0J603541M | `/dev/disk/by-id/ata-Samsung_SSD_850_PRO_1TB_S3D2NX0J603541M` | `pv00`, `boot`, `boot_efi` |
| 2 | Samsung SSD 850 PRO 1TB | S3D2NX0J603337T | `/dev/disk/by-id/ata-Samsung_SSD_850_PRO_1TB_S3D2NX0J603337T` | `pv00`, `boot`, `boot_efi`. Suspect: dropped off the controller once, 2026-10-23 |
| 3 | Seagate ST1000DM010-2EP102 | W9A63479 | `/dev/disk/by-id/ata-ST1000DM010-2EP102_W9A63479` | `hdd` slot 0 |
| 4 | Seagate ST1000DM010-2EP102 | W9A7NB9T | `/dev/disk/by-id/ata-ST1000DM010-2EP102_W9A7NB9T` | `hdd` slot 1 |
| 5 | Seagate ST1000DM010-2EP102 | W9A9ZDXL | `/dev/disk/by-id/ata-ST1000DM010-2EP102_W9A9ZDXL` | `hdd` slot 2 |

Notes:

- Each disk also has `scsi-SATA_...` and `wwn-0x...` names under
  `/dev/disk/by-id/`. The `ata-` names carry the full model and serial and
  are the ones used in this repository.
- ADR-0008 describes the 870 EVO as given whole to the RAID5 array. In
  fact it carries a single partition and that partition is the member.
- The 850 PROs are 953.9 GiB devices; the 870 EVO and the Seagates are
  931.5 GiB.

Controller: Dell PERC H330 Mini (LSI SAS3008, PCI `02:00.0`, IDs
`1000:005f` / `1028:1f4b`), HBA mode, driver `megaraid_sas`. No cache, no
battery. Does not pass TRIM to the 850 PROs.

## Arrays

| Name | Kernel (build day) | Level | Metadata | Chunk | Consistency | Size | Array UUID |
|---|---|---|---|---|---|---|---|
| `boot_efi` | md125 | RAID1 | 1.0 | — | bitmap | 600.9 MiB | `423d96c7:4ef8f42f:f3557aae:b2556f8b` |
| `boot` | md126 | RAID1 | 1.2 | — | bitmap | 1023 MiB | `0869e761:61d77346:802af477:b766ff67` |
| `pv00` | md127 | RAID5 | 1.2 | 512K | bitmap | 1862.8 GiB | `cbd4af0f:a09c5e3b:eaac2317:a3411362` |
| `hdd` | md124 | RAID5 | 1.2 | 512K | PPL | 1862.77 GiB | `33138201:5f40ed03:602fccaf:42e9527a` |

Names are stored as `r730xd-01:<name>` and appear as `/dev/md/<name>`.
`/etc/mdadm.conf` has one `ARRAY` line for each.

Fault tolerance: each array survives the loss of one member. Bays 1 and 2
are each in three arrays, so losing one of those disks degrades all three
at once. Losing both 850 PROs loses `/boot`, `/boot/efi` and `pv00`.

## LVM

| VG | PV | PV UUID | Size | Free |
|---|---|---|---|---|
| `ssd-vg` | `/dev/md/pv00` | `6NWndo-eS7c-lv7G-w3Mm-I6lF-d38N-nDe2DC` | about 1.82 TiB | about 180 GiB (measured: ________) |
| `hdd-vg` | `/dev/md/hdd` | `F1ulUJ-CeG1-phvu-LsoV-OnPt-xHZX-AUvYFD` | <1.82 TiB | <388.77 GiB |

| LV | Type | Size | Filesystem | FS UUID | Mount |
|---|---|---|---|---|---|
| `ssd-vg/root` | thick | 50 GiB | xfs | `04e88ab7-9c9f-4f3e-ac41-3c9ab6eb1ce5` | `/` |
| `ssd-vg/swap` | thick | 8 GiB | swap | `678ea59e-1862-4b2c-98db-93127716a86c` | swap |
| `ssd-vg/vmpool` | thin pool, 512K chunk, zeroing on | about 1.6 TiB (metadata 204 MiB) | — | — | holds VM disks as thin LVs |
| `hdd-vg/backups` | thick | 1 TiB | xfs | `3a8fc48c-24bb-4b62-936e-f25c0f1a5d73` | `/srv/backups` |
| `hdd-vg/bulk` | thick | 300 GiB | xfs | `723273be-691e-430f-a5dd-d11c5242eb9d` | `/srv/bulk` |
| `hdd-vg/isos` | thick | 150 GiB | xfs | `be3e0043-beea-47b0-8876-e708f9b45ee9` | `/var/lib/libvirt/isos` |

Other filesystems: `/boot` xfs `47ccd58d-cd75-41b8-84b2-9486c64176bf` on
`boot`; `/boot/efi` vfat `0C24-0FD5` on `boot_efi`.

The XFS filesystems on `hdd-vg` are aligned to the array: stripe unit
512 KiB, stripe width 1 MiB.

## What lives where

| Path | On | Contents |
|---|---|---|
| `/` | `ssd-vg/root` | Operating system |
| `/dev/ssd-vg/<vm-disk>` | thin LV in `vmpool` | One per VM disk, attached raw |
| `/srv/backups` | `hdd-vg/backups` | Reserved for the backup repository (Module 1.6) |
| `/srv/bulk` | `hdd-vg/bulk` | Bulk data |
| `/srv/bulk/repo/rhel9.5` | `hdd-vg/bulk` | Offline BaseOS and AppStream repositories, about 12.7 GB |
| `/var/lib/libvirt/isos` | `hdd-vg/isos` | Install images; libvirt pool `isos` |

## Capacity summary

| Tier | Raw | Usable | Allocated | Unallocated |
|---|---|---|---|---|
| SSD | 3 × 1 TB | about 1.82 TiB | root 50 GiB, swap 8 GiB, vmpool about 1.6 TiB | about 180 GiB |
| HDD | 3 × 1 TB | about 1.82 TiB | 1474 GiB | about 388 GiB |

## Known limits

- Single host, no UPS (ADR-0007).
- `pv00` has no write-hole protection yet (ADR-0011).
- No TRIM to the SSD tier (ADR-0014).
- Seagate disks are desktop models without error-recovery control
  (ADR-0013).
- Thin pool exhaustion stops all VMs (ADR-0014).
