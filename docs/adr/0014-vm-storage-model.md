# ADR-0014: VM disk storage model

- Status: Accepted
- Date: 2026-11-09
- Module: 0.2
- Related: ADR-0008 (disk layout), ADR-0011

## Context

VM disks go on the SSD tier (`ssd-vg`, on the `pv00` RAID5 array). The
project depends on cheap snapshots: every security-range exercise
snapshots its targets before and rolls back after, and risky host changes
should be preceded by a snapshot of root.

After the install, `ssd-vg` held `root` (50 GiB, thick) and `swap` (8 GiB),
with the rest unallocated.

libvirt can manage an LVM volume group as a storage pool, but that pool
type creates only ordinary thick volumes. It has no way to create thin
volumes or thin snapshots.

## Options

| Option | Snapshots | Problem |
|---|---|---|
| qcow2 image files on a filesystem | qcow2's own | A second copy-on-write layer on top of LVM; ruled out by the charter |
| libvirt logical pool on `ssd-vg` | None usable | Thick volumes only |
| Hand-made thin LVs attached as raw block devices | LVM thin snapshots | VM disks are created with `lvcreate`, outside libvirt |

## Decision

1. One thin pool, `ssd-vg/vmpool`, created with `-l 90%FREE` and a 512 KiB
   chunk size. About 1.6 TiB.
2. Each VM disk is a thin LV in that pool
   (`lvcreate --type thin --thinpool vmpool -V <SIZE> -n <NAME> ssd-vg`),
   attached to the VM as a raw block device, `/dev/ssd-vg/<NAME>`.
3. libvirt manages no pool for VM disks. It has one `dir` pool, `isos`, at
   `/var/lib/libvirt/isos` (the `hdd-vg/isos` filesystem), defined in
   `qemu:///system`.
4. `root` stays a thick volume outside the pool.
5. About 180 GiB of `ssd-vg` stays unallocated for snapshots of `root`,
   which, being thick, need their own space.
6. Pool zeroing stays enabled.

## Consequences

- Snapshot and rollback of any VM disk is an LVM operation and is
  instantaneous.
- Over-provisioning is possible. If the pool's data or metadata reaches
  100%, writes to every VM stall. Until Module 1.5 adds an alert, `Data%`
  and `Meta%` are checked by hand. Working ceiling: act at 80%.
- A full pool does not stop the host, because root is not in it.
- There is no spare room for pool auto-extension without spending the
  root-snapshot reserve. Auto-extension is left off.
- Zeroing slows first writes to new chunks (LVM warns about this with
  512 KiB chunks). Kept so a new VM disk can never read a deleted VM's
  blocks, which matters once the range shares the pool.
- **No TRIM.** `lsblk -D` reports `DISC-MAX 0B` for both 850 PRO disks and
  for the array; only the 870 EVO accepts discards through this
  controller. Discards therefore never reach the SSDs. `fstrim` on this
  tier is a no-op and the SSDs rely on their own garbage collection.
- VM definitions will reference `/dev/ssd-vg/<NAME>` directly. Tooling in
  Lane C (Terraform libvirt provider) must be written for that, not for
  libvirt-managed volumes.

## What would change this decision

- A second hypervisor host: shared or replicated storage replaces local
  thin LVs.
- A need to move VM disks around as files: qcow2 on a filesystem.
- Evidence that zeroing is the bottleneck when provisioning many VMs:
  `lvchange -Zn ssd-vg/vmpool`, recorded in a superseding ADR.
