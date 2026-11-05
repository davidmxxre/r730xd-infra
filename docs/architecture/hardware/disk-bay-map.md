# Disk bay map — r730xd-01

**Never trust `/dev/sdX` names; they change between boots.** Identify disks by
serial number.

## Bay numbering on this chassis

Facing the front, the 12 × 3.5" bays are numbered **down each column, left to
right**. The iDRAC bay number equals the physical slot.

```
        col 1   col 2   col 3   col 4
top      [0]     [3]     [6]     [9]
middle   [1]     [4]     [7]     [10]
bottom   [2]     [5]     [8]     [11]
```

Neither drive LED helps with identification on this server in HBA mode: the
locate LED ignores `racadm storage blink`, and the activity LED does not flicker
under `dd` reads. The positions below were confirmed by drive model and position
on 2026-11-04 (SSDs in column 1, Barracudas in column 2, the 870 EVO top-left),
matching the iDRAC bay numbers.

To check a drive's serial physically: shut down cleanly, pull **one** caddy, read
the label, and return it to the **same slot** before pulling the next.

| Bay FQDD (`Disk.Bay.N:Enclosure.Internal.0-1:RAID.Integrated.1-1`) | Physical slot | Model | Serial | Size | `sdX` at first boot | Role | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Disk.Bay.0 | Col 1, top | Samsung SSD 870 EVO 1TB | S75BNL0X441461N | 931.5 GiB | sdb | `md127` RAID5 member | No boot partitions |
| Disk.Bay.1 | Col 1, middle | Samsung SSD 850 PRO 1TB | S3D2NX0J603541M | 953.9 GiB | sde | RAID5, `/boot`, ESP | |
| Disk.Bay.2 | Col 1, bottom | Samsung SSD 850 PRO 1TB | S3D2NX0J603337T | 953.9 GiB | sdc | RAID5, `/boot`, ESP | **Suspect**: dropped once 2026-10-23 (PDR87 reset ×2, PDR3, PDR5 removed) |
| Disk.Bay.3 | Col 2, top | Seagate Barracuda ST1000DM010 | W9A63479 | 931.5 GiB | sda | Future `hdd-vg` | Desktop drive |
| Disk.Bay.4 | Col 2, middle | Seagate Barracuda ST1000DM010 | W9A7NB9T | 931.5 GiB | sdd | Future `hdd-vg` | Desktop drive |
| Disk.Bay.5 | Col 2, bottom | Seagate Barracuda ST1000DM010 | W9A9ZDXL | 931.5 GiB | sdf | Future `hdd-vg` | Desktop drive |

Bays 6–11 are empty.

## Persistent names

On the host, the stable names are:

```
ls -l /dev/disk/by-id/ | grep -E 'ata-|wwn-' | grep -v part
```

`ls -l` shows each by-id link and the `sdX` it currently points to. Use the
`ata-<model>_<serial>` names in mdadm, fstab and scripts.

## Mounting notes

- Drives must sit with the connector end **flush with the caddy's back edge**.
  A drive mounted short never reaches the backplane even when the caddy latches.
- 2.5" SSDs use a 2.5"→3.5" adapter that positions the connector where a 3.5"
  drive's would be.

## Open items

- Bay 2 SSD: SMART long test, a full read test, and `dmesg`/LC log check in the
  host OS module. If it drops again, move it to an empty bay; if the fault follows
  the drive, replace it.
- Barracudas lack time-limited error recovery: the storage module sets a 180 s
  SCSI timeout for them so one bad sector doesn't eject a disk from the array.
