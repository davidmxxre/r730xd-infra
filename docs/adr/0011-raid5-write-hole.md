# ADR-0011: RAID5 write-hole mitigation

- Status: Accepted for the `hdd` array. Open for `pv00` (see Follow-up)
- Date: 2026-11-09
- Module: 0.2
- Related: ADR-0007 (UPS, proposed), ADR-0008 (disk layout), ADR-0012

## Context

Both storage tiers are three-disk mdadm RAID5. A RAID5 stripe update writes
a data chunk and a parity chunk to different disks. If power is lost
between the two writes, data and parity disagree. Nothing detects this
while all disks are present. If a disk later fails, that stripe is rebuilt
from wrong parity and returns corrupt data without an error.

The host has no UPS (ADR-0007 is still proposed) and the controller, a PERC
H330 Mini in HBA mode, has no cache or battery. Unclean power loss is a
realistic event. The SMART data of one SSD already records 30 power-loss
recoveries from its previous life.

The installer created `pv00` with an internal write-intent bitmap. A bitmap
shortens the resync after a crash. It does not close the write hole.

## Options

| Option | Closes the hole | Cost |
|---|---|---|
| Write-intent bitmap | No | Small write overhead |
| Partial Parity Log (PPL) | Yes | Slower partial-stripe writes; cannot coexist with a bitmap |
| Dedicated journal device | Yes | Needs a spare fast device, which is a single point of failure unless mirrored. None is available |
| UPS with clean shutdown | Removes the trigger | Hardware purchase |

## Decision

New arrays on this host are created with `--consistency-policy=ppl` for as
long as there is no UPS. The `hdd` array was created this way.

## Consequences

- `hdd` is protected against the write hole.
- Writes to `hdd` are slower than with a bitmap. Acceptable for a tier
  that holds backups, ISOs and the package repository.
- After an unclean shutdown `hdd` recovers from the log; no full resync is
  required.
- A journal device is ruled out until there is spare hardware.

## Follow-up

`pv00` carries the root filesystem and will carry every VM disk. It still
has a bitmap and no PPL. One of the following must happen before VMs hold
data that matters:

1. Convert `pv00` to PPL (runbook, "Convert pv00 to PPL"), accepting slower
   writes on the SSD tier; or
2. Accept ADR-0007, install a UPS configured to shut the host down cleanly,
   and record here that the bitmap is kept deliberately.

## What would change this decision

A UPS with tested clean shutdown. At that point the bitmap becomes
acceptable on both arrays and PPL can be removed from `hdd` to regain write
speed (`mdadm --grow --consistency-policy=resync`, then
`--bitmap=internal`).
