# ADR-0012: RAID level for the HDD tier

- Status: Accepted
- Date: 2026-11-09
- Module: 0.2
- Related: ADR-0011, ADR-0013

## Context

Three Seagate ST1000DM010 1 TB disks (bays 3, 4, 5) were unused after the
install. The tier is for backups, ISO images, the offline package
repository, bulk data and, from Phase 6, packet captures and log
retention. Everything planned for it is either a copy of data that lives
elsewhere or can be re-created.

The disks are desktop models without error-recovery control.

## Options

| Option | Usable | Tolerates | Notes |
|---|---|---|---|
| RAID5 | about 1.82 TiB | one disk | Rebuild reads both survivors in full |
| RAID1, three-way | about 0.91 TiB | two disks | Half the capacity |
| RAID10 | — | — | Needs four disks |

## Decision

RAID5 across the three whole disks, 512 KiB chunk, metadata 1.2, name
`hdd`, consistency policy PPL (ADR-0011).

Created 2026-11-09. Array UUID `33138201:5f40ed03:602fccaf:42e9527a`.

## Consequences

- 1862.77 GiB usable.
- One disk can fail. During the rebuild there is no redundancy, and a
  rebuild is when a latent unreadable sector on a desktop disk is most
  likely to surface. ADR-0013 (timeout and weekly scrub) exists to reduce
  that risk.
- This tier must not hold the only copy of anything. When backups land
  here in Module 1.6, the thing being backed up is on the SSD tier; an
  off-host copy is still required for the charter's "RAID is not a
  backup" rule to hold.

## What would change this decision

- A fourth disk: move to RAID10 or RAID6.
- The tier becoming the sole home of irreplaceable data: mirror instead.
- Replacing the disks with NAS or enterprise models removes the main
  rebuild risk but not the single-disk tolerance.
