# Runbook: r730xd-01 storage, virtualization and disk monitoring

Module 0.2. This document is meant to be used cold. It gives commands,
what to expect, and what to do when the result is different. Explanations
of why are in `docs/modules/0.2/` and ADRs 0011-0015.

## 1. Facts

| | |
|---|---|
| Host | `r730xd-01`, RHEL 9.5, kernel 5.14.0-503.11.1.el9_5, UEFI, Secure Boot on |
| Access | iDRAC serial console (see `docs/onboarding/host-access-and-packages.md`). Not on any LAN |
| Login | `dave`, with `sudo`. Root is locked |
| Controller | PERC H330 Mini, HBA mode, driver `megaraid_sas`, PCI `02:00.0` |
| Network time | Off. Clock set by hand |
| Packages | Local repo at `/srv/bulk/repo/rhel9.5` |

Arrays (use the name, never the number):

| Name | Level | Members | UUID | Used for |
|---|---|---|---|---|
| `boot_efi` | RAID1, meta 1.0 | partition 3 of both 850 PROs | `423d96c7:4ef8f42f:f3557aae:b2556f8b` | `/boot/efi` |
| `boot` | RAID1, meta 1.2 | partition 2 of both 850 PROs | `0869e761:61d77346:802af477:b766ff67` | `/boot` |
| `pv00` | RAID5, 512K, bitmap | partition 1 of all three SSDs | `cbd4af0f:a09c5e3b:eaac2317:a3411362` | PV of `ssd-vg` |
| `hdd` | RAID5, 512K, PPL | three Seagate disks, whole | `33138201:5f40ed03:602fccaf:42e9527a` | PV of `hdd-vg` |

Confirm the links exist with `ls -l /dev/md/`.

Full disk and volume map: `docs/architecture/storage-layout.md`.

Rules:

- Never use `/dev/sdX` in a command that writes. Use
  `/dev/disk/by-id/ata-<MODEL>_<SERIAL>`.
- Writing to a root-owned file: `... | sudo tee FILE`, not
  `sudo ... > FILE`.
- `virsh` must report `qemu:///system` from `virsh uri` before you define
  anything.

## 2. Health check (run any time)

```
cat /proc/mdstat
sudo lvs -o lv_name,lv_size,data_percent,metadata_percent ssd-vg
df -hT / /srv/backups /srv/bulk /var/lib/libvirt/isos
journalctl -t disk-alert --since "-7 days"
systemctl --failed
```

Healthy: four arrays with all `U`; `vmpool` data and metadata below 80%;
the three `/srv` and libvirt paths on `hdd--vg-*` devices; no unexpected
`disk-alert` lines; no failed units.

## 3. Build procedure (as performed 2026-11-09)

Run in order. Each step: command, expected, check.

### 3.1 Pre-checks

```
lspci -nnk -d ::0104
cat /proc/mdstat
lsblk -d -o NAME,SERIAL,SIZE,MODEL,ROTA
for s in W9A63479 W9A7NB9T W9A9ZDXL; do
  d=/dev/disk/by-id/ata-ST1000DM010-2EP102_$s
  echo "== $d"; sudo wipefs "$d"; sudo mdadm --examine "$d"
done
```

Expect: `megaraid_sas`; three healthy arrays; six disks; for each Seagate,
no `wipefs` output and `No md superblock detected`. Stop if any Seagate is
not blank.

### 3.2 HDD array

```
sudo mdadm --create /dev/md/hdd --name=hdd --level=5 --raid-devices=3 \
  --chunk=512 --consistency-policy=ppl \
  /dev/disk/by-id/ata-ST1000DM010-2EP102_W9A63479 \
  /dev/disk/by-id/ata-ST1000DM010-2EP102_W9A7NB9T \
  /dev/disk/by-id/ata-ST1000DM010-2EP102_W9A9ZDXL
sudo mdadm --detail /dev/md/hdd
```

Record every array in `/etc/mdadm.conf`. The installer did not write
`ARRAY` lines for the boot arrays (they assemble from `rd.md.uuid=` on
the kernel command line). This adds only lines that are missing, so it
is safe to repeat:

```
sudo cp -a /etc/mdadm.conf /etc/mdadm.conf.pre-0.2
cat /etc/mdadm.conf
sudo mdadm --detail --scan | while read -r line; do
  uuid=${line##*UUID=}
  grep -q "$uuid" /etc/mdadm.conf || echo "$line" | sudo tee -a /etc/mdadm.conf
done
grep '^ARRAY' /etc/mdadm.conf
```

Then check whether the initramfs carries its own copy:

```
lsinitrd -f etc/mdadm.conf
```

If it prints a file that differs from `/etc/mdadm.conf`:

```
sudo cp -a /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.pre-mdadm
sudo dracut -f
lsinitrd -f etc/mdadm.conf
```

Fallback if the next boot fails: the GRUB "rescue" entry, then restore the
`.pre-mdadm` image.

Expect: `array /dev/md/hdd started.`; detail shows `raid5`,
`Consistency Policy : ppl`, and at first `clean, degraded, recovering`
with one `spare rebuilding` (normal for a new RAID5, about two hours).
Check: `grep -c '^ARRAY' /etc/mdadm.conf` prints `4`.

### 3.3 Volume group, volumes, filesystems, mounts

```
sudo pvcreate /dev/md/hdd
sudo vgcreate hdd-vg /dev/md/hdd
sudo lvcreate -L 1T   -n backups hdd-vg
sudo lvcreate -L 300G -n bulk    hdd-vg
sudo lvcreate -L 150G -n isos    hdd-vg
for lv in backups bulk isos; do sudo mkfs.xfs /dev/hdd-vg/$lv; done
sudo mkdir -p /srv/backups /srv/bulk /var/lib/libvirt/isos
sudo blkid /dev/hdd-vg/backups /dev/hdd-vg/bulk /dev/hdd-vg/isos
sudo cp -a /etc/fstab /etc/fstab.pre-0.2
sudo vim /etc/fstab
```

Lines added to `/etc/fstab` (UUIDs from `blkid`; these are the built
values):

```
UUID=3a8fc48c-24bb-4b62-936e-f25c0f1a5d73 /srv/backups           xfs defaults 0 0
UUID=723273be-691e-430f-a5dd-d11c5242eb9d /srv/bulk              xfs defaults 0 0
UUID=be3e0043-beea-47b0-8876-e708f9b45ee9 /var/lib/libvirt/isos  xfs defaults 0 0
```

```
sudo findmnt --verify
sudo systemctl daemon-reload
sudo mount -a
df -hT /srv/backups /srv/bulk /var/lib/libvirt/isos
sudo vgs hdd-vg
```

Expect: no verify errors; three `hdd--vg-*` devices; `VFree` about
388 GiB.

Recommended change, not applied on the build day: in those three lines
replace `defaults` with `defaults,nofail`, then run
`sudo findmnt --verify` and `sudo systemctl daemon-reload`. Without
`nofail`, a failed HDD array stops the boot, and because root is locked
there is no emergency shell to fix it from. With it, the host boots and
the three paths are simply empty until the array is repaired. Applied:
yes / no, date: ________

### 3.4 Offline repository

Insert the RHEL 9.5 install USB.

```
lsblk -f
sudo mkdir -p /mnt/rhel-dvd
sudo mount -o ro <USB_ISO_PARTITION> /mnt/rhel-dvd
findmnt /srv/bulk
sudo mkdir -p /srv/bulk/repo/rhel9.5
sudo cp -a /mnt/rhel-dvd/BaseOS /mnt/rhel-dvd/AppStream /srv/bulk/repo/rhel9.5/
sudo umount /mnt/rhel-dvd
```

`<USB_ISO_PARTITION>` is the `iso9660` partition labelled
`RHEL-9-5-0-BaseOS-x86_64` in the `lsblk -f` output (it was `/dev/sdg1`).
`findmnt /srv/bulk` must show `/dev/mapper/hdd--vg-bulk` before the copy.

```
sudo tee /etc/yum.repos.d/rhel-dvd.repo >/dev/null <<'EOF'
[dvd-baseos]
name=RHEL9.5 DVD BaseOS
baseurl=file:///srv/bulk/repo/rhel9.5/BaseOS
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[dvd-appstream]
name=RHEL9.5 DVD AppStream
baseurl=file:///srv/bulk/repo/rhel9.5/AppStream
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
sudo dnf clean all
sudo dnf repolist -v | grep -E 'Repo-id|Repo-pkgs'
```

Expect: two repositories with package counts. "Not registered" notices
are normal.

### 3.5 Thin pool

```
sudo lvcreate --type thin-pool --name vmpool -l 90%FREE --chunksize 512K ssd-vg
sudo lvs -a ssd-vg
sudo vgs ssd-vg
```

Expect: two warnings about zeroing and chunk size (advice only);
`vmpool` about 1.6 TiB; `VFree` about 180 GiB.
Measured `VFree`: ____________

### 3.6 Virtualization

```
sudo dnf group install "Virtualization Host"
systemctl is-enabled libvirtd.service virtqemud.socket virtstoraged.socket
```

If `libvirtd.service` is `enabled`, switch to the modular daemons:

```
sudo systemctl disable --now libvirtd.service libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket
for drv in qemu interface network nodedev nwfilter secret storage; do
  sudo systemctl unmask virt${drv}d.service virt${drv}d.socket virt${drv}d-ro.socket virt${drv}d-admin.socket
  sudo systemctl enable virt${drv}d.service virt${drv}d.socket virt${drv}d-ro.socket virt${drv}d-admin.socket
  sudo systemctl start  virt${drv}d.socket virt${drv}d-ro.socket virt${drv}d-admin.socket
done
```

```
virt-host-validate qemu
sudo usermod -aG libvirt dave
echo 'export LIBVIRT_DEFAULT_URI=qemu:///system' >> ~/.bashrc
```

Log out and in.

```
virsh uri
virsh pool-define-as isos dir --target /var/lib/libvirt/isos
virsh pool-start isos
virsh pool-autostart isos
virsh pool-list --all
```

Expect: first three `virt-host-validate` lines `PASS` (warnings for
cgroup 'devices', IOMMU and secure guest are expected);
`qemu:///system`; `isos active yes`.

### 3.7 Seagate timeout and scrub

```
sudo tee /etc/udev/rules.d/60-raid-disk-timeout.rules >/dev/null <<'EOF'
ACTION=="add|change", SUBSYSTEM=="block", KERNEL=="sd[a-z]", ATTRS{model}=="ST1000DM010*", ATTR{device/timeout}="180"
EOF
sudo udevadm control --reload
sudo udevadm trigger --action=change --subsystem-match=block
for s in W9A63479 W9A7NB9T W9A9ZDXL; do
  d=$(basename "$(readlink -f /dev/disk/by-id/ata-ST1000DM010-2EP102_$s)")
  echo "$s $d $(cat /sys/block/$d/device/timeout)"
done
sudo systemctl enable --now raid-check.timer
```

Expect: three lines ending `180`. If any shows `90`, the rule did not
parse: `sudo udevadm test /sys/block/<SEAGATE_SDX> 2>&1 | grep -iE 'timeout|60-raid|invalid'`,
where `<SEAGATE_SDX>` is the middle column of the loop output.

### 3.8 Alerts and SMART

```
sudo tee /usr/local/sbin/disk-alert.sh >/dev/null <<'EOF'
#!/bin/bash
# Called by mdadm as: disk-alert.sh EVENT ARRAY [MEMBER]
# Called by smartd with details in SMARTD_* environment variables
logger -t disk-alert -p daemon.err "event=${1:-smartd} array=${2:-} dev=${3:-${SMARTD_DEVICE:-}} msg=${SMARTD_MESSAGE:-}"
EOF
sudo chmod 755 /usr/local/sbin/disk-alert.sh

echo "PROGRAM /usr/local/sbin/disk-alert.sh" | sudo tee -a /etc/mdadm.conf
sudo systemctl restart mdmonitor

sudo cp -a /etc/smartmontools/smartd.conf /etc/smartmontools/smartd.conf.pre-0.2
sudo sed -i 's/^DEVICESCAN/#DEVICESCAN/' /etc/smartmontools/smartd.conf
ls /dev/disk/by-id/ata-* | grep -v part \
  | sed 's#$# -d sat -a -o on -S on -s (S/../.././02|L/../../6/03) -m <nomailer> -M exec /usr/local/sbin/disk-alert.sh#' \
  | sudo tee -a /etc/smartmontools/smartd.conf
sudo systemctl enable --now smartd mdmonitor
sudo systemctl restart smartd
```

`<nomailer>` is typed literally. Run each `tee -a` once only.

```
sudo mdadm --monitor --scan --test --oneshot
journalctl -t disk-alert --since "-2 min"
journalctl -u smartd -b | grep Monitoring
```

Expect: four `event=TestMessage` lines; `Monitoring 6 ATA/SATA`.

## 4. Validation

Fill in on the build day and again on the cold run.

| # | Check | Command | Pass | Build result / date | Cold-run result / date |
|---|---|---|---|---|---|
| 1 | Arrays healthy | `cat /proc/mdstat` | four arrays, all `U`, no progress line | | |
| 2 | HDD array policy | `sudo mdadm --detail /dev/md/hdd \| grep -E 'State \|Policy'` | `clean`, `ppl` | | |
| 3 | Array recorded | `grep -c '^ARRAY' /etc/mdadm.conf` | `4` | | |
| 4 | Mounts | `df -hT /srv/backups /srv/bulk /var/lib/libvirt/isos` | three `hdd--vg-*` | | |
| 5 | fstab | `sudo findmnt --verify` | no errors | | |
| 6 | Repo | `sudo dnf clean all && sudo dnf repolist -v \| grep -E 'Repo-id\|Repo-pkgs'` | two repos, counts > 0 | | |
| 7 | Pool and reserve | `sudo lvs ssd-vg; sudo vgs ssd-vg` | `vmpool`; `VFree` about 180g | | |
| 8 | Thin volume | section 5.1 below, create and remove `vm-scratch` | works | | |
| 9 | Acceleration | `virt-host-validate qemu \| head -3` | three `PASS` | | |
| 10 | libvirt layout | `systemctl is-active libvirtd.service virtqemud.socket` | `inactive`, `active` | | |
| 11 | virsh | `virsh uri; virsh pool-list --all` | `qemu:///system`; `isos active yes` | | |
| 12 | Timeouts | loop in 3.7 | three × `180` | | |
| 13 | Scrub | section 5.5 below | `mismatch_cnt` = `0` | | |
| 14 | Scrub timer | `systemctl is-enabled raid-check.timer` | `enabled` | | |
| 15 | RAID alerts | `sudo mdadm --monitor --scan --test --oneshot; journalctl -t disk-alert --since "-2 min"` | four TestMessage lines | | |
| 16 | SMART daemon | `journalctl -u smartd -b \| grep Monitoring` | 6 devices | | |
| 17 | Disk health | section 5.7 below | six `PASSED`, raw counts 0 | | |
| 18 | Reboot | reboot; repeat 1, 4, 6, 11, 12 | all pass | | |

## 5. Common operations

### 5.1 Create and remove a VM disk

```
sudo lvcreate --type thin --thinpool vmpool -V <SIZE> -n <VM_DISK_NAME> ssd-vg
ls -l /dev/ssd-vg/<VM_DISK_NAME>
sudo lvremove ssd-vg/<VM_DISK_NAME>
```

`<SIZE>` like `40G`. `<VM_DISK_NAME>` follows the naming convention from
Module 0.3. For validation use `-V 40G -n vm-scratch`.

### 5.2 Snapshot and roll back a VM disk

```
sudo lvcreate -s -n <VM_DISK_NAME>-pre ssd-vg/<VM_DISK_NAME>
# ... do the risky thing ...
virsh shutdown <VM_NAME>                      # the VM must be off to roll back
sudo lvconvert --merge ssd-vg/<VM_DISK_NAME>-pre   # roll back, snapshot is consumed
# or, to keep the changes:
sudo lvremove ssd-vg/<VM_DISK_NAME>-pre
```

### 5.3 Snapshot root before a risky host change

```
sudo lvcreate -s -L 20G -n root-pre ssd-vg/root
# to roll back:
sudo lvconvert --merge ssd-vg/root-pre && sudo reboot
# to keep the change:
sudo lvremove ssd-vg/root-pre
```

The snapshot fills as root changes. If it reaches 100% it becomes invalid;
check with `sudo lvs ssd-vg`. Do not leave one in place for days.

### 5.4 Grow a volume on hdd-vg

```
sudo vgs hdd-vg
sudo lvextend -r -L +<SIZE> hdd-vg/<LV_NAME>
```

`-r` grows the XFS filesystem too. XFS cannot be shrunk.

### 5.5 Scrub the HDD array by hand

```
cat /proc/mdstat
MD=$(basename "$(readlink -f /dev/md/hdd)"); echo "$MD"
echo check | sudo tee /sys/block/$MD/md/sync_action
cat /proc/mdstat
```

Wait until `cat /sys/block/$MD/md/sync_action` prints `idle` (about two
hours), then:

```
cat /sys/block/$MD/md/mismatch_cnt
```

Pass: `0`. `Device or resource busy` on the `tee` means a rebuild or
check is already running. To stop a check:
`echo idle | sudo tee /sys/block/$MD/md/sync_action`. Stay in the same
shell; `$MD` is lost if you switch to a root shell.

### 5.6 Check the thin pool

```
sudo lvs -o lv_name,lv_size,data_percent,metadata_percent ssd-vg
```

Act at 80% on either figure: remove snapshots and unused volumes first.

### 5.7 Check disk health

```
ls /dev/disk/by-id/ata-* | grep -v part | while read -r d; do
  echo "== $d"
  sudo smartctl -H -A -d sat "$d" | grep -E 'result|Reallocated_Sector|Pending|Uncorrect|CRC|Wear_Level'
done
```

### 5.8 Install a package

```
findmnt /srv/bulk
sudo dnf install <PACKAGE>
```

### 5.9 Replace a failed Seagate disk in `hdd`

```
sudo mdadm --detail /dev/md/hdd                 # identify the failed member and its serial
sudo mdadm /dev/md/hdd --fail <FAILED_DISK_BY_ID> --remove <FAILED_DISK_BY_ID>
# replace the physical disk in the bay recorded in storage-layout.md
ls /dev/disk/by-id/ | grep -v part              # find the new disk's by-id name
sudo wipefs <NEW_DISK_BY_ID>                     # must print nothing
sudo mdadm /dev/md/hdd --add <NEW_DISK_BY_ID>
cat /proc/mdstat                                 # rebuild progress
```

If the disk has vanished completely, `--fail` and `--remove` may report it
is not there; use `sudo mdadm /dev/md/hdd --remove detached`. If the new
disk is a different model, update the udev rule in 3.7 and the `smartd.conf`
line. Update `storage-layout.md`.

**Replacing an SSD is not covered yet.** The two 850 PROs each hold three
partitions in three arrays plus an EFI boot entry. That procedure must be
written and tested before it is needed. Tracked as a gap.

## 6. Convert `pv00` to PPL (ADR-0011 follow-up, not yet done)

Do this with the host idle, no VMs running, and the array clean.

```
ls -l /dev/md/
sudo mdadm --detail /dev/md/pv00 | grep -E 'State |Policy'
sudo mdadm --grow /dev/md/pv00 --bitmap=none
sudo mdadm --grow /dev/md/pv00 --consistency-policy=ppl
sudo mdadm --detail /dev/md/pv00 | grep Policy
```

Expect before: `clean`, `bitmap`. After: `ppl`.

If the second `--grow` is refused, nothing has been lost; put the bitmap
back with `sudo mdadm --grow /dev/md/pv00 --bitmap=internal`. This
conversion has not been tried on this array; treat the first attempt as a
change with a rollback, not a routine step.

To reverse later: `--consistency-policy=resync`, then `--bitmap=internal`.

## 7. Suspect disk: bay 2, Samsung 850 PRO S3D2NX0J603337T

Dropped off the controller once on 2026-10-23. It is a member of
`boot_efi`, `boot` and `pv00`. SMART on 2026-11-09: PASSED, 0 reallocated,
0 CRC errors, wear indicator 98, 22718 power-on hours.

```
sudo smartctl -t long -d sat /dev/disk/by-id/ata-Samsung_SSD_850_PRO_1TB_S3D2NX0J603337T
sudo smartctl -l selftest -d sat /dev/disk/by-id/ata-Samsung_SSD_850_PRO_1TB_S3D2NX0J603337T
sudo dmesg -T | grep -iE 'megaraid|sd[a-z]|reset|offline' | tail -50
```

The long test takes about 8.5 hours. Result: ____________ Date: ________

If it drops again: check `/proc/mdstat`. Three arrays will be degraded.
Do not reboot to "see if it comes back" before reading the iDRAC lifecycle
log. Re-add with `sudo mdadm /dev/md/<ARRAY_NAME> --re-add <PARTITION_BY_ID>`
for each of the three partitions once the disk is visible again.

## 8. When something is wrong

| Symptom | Likely cause | Action |
|---|---|---|
| `dnf` fails with `Curl error (37): Couldn't read a file://` | `/srv/bulk` not mounted | `findmnt /srv/bulk`; `sudo mount -a`; then check the `hdd` array |
| `dnf`: `Empty mirrorlist and no basepath specified` | malformed `baseurl` | rewrite the repo file from 3.4 |
| Boot stops with `Cannot open access to console, the root account is locked` | an `/etc/fstab` entry cannot be mounted, probably `hdd` did not assemble. systemd wants the emergency shell, and root has no password | boot the install USB, choose Troubleshooting then Rescue, mount the installed system, and either fix the array or comment out the three `hdd-vg` lines in `/etc/fstab`. Prevented by the `nofail` change in 3.3 |
| `/proc/mdstat` shows `[UU_]` or `(F)` | member failed or ejected | `sudo mdadm --detail /dev/md/<ARRAY_NAME>`; `journalctl -k -b \| grep -iE 'sd[a-z]\|md1'`; SMART on that disk; 5.9 |
| VMs frozen, host responsive | thin pool data or metadata at 100% | 5.6; free space by removing snapshots; VMs resume when writes can complete |
| `virsh list --all` empty but VMs exist | talking to `qemu:///session` | `virsh uri`; `source ~/.bashrc` |
| `virsh`: `authentication unavailable` | not in group `libvirt` in this login | `id -nG`; log out and in |
| ISO pool shows about 50 GiB capacity | `hdd-vg/isos` not mounted; pool is on root | mount it; `virsh pool-refresh isos` |
| Seagate timeout back at `90` | rule file missing or broken | 3.7 |
| `Permission denied` writing under `/sys` or `/etc` with `sudo` | redirect ran as dave | use `\| sudo tee` |
| `/sys/block//md/...: No such file or directory` | empty shell variable | set it again in the current shell |
| No `disk-alert` lines after the test | `PROGRAM` line in the wrong file, script not executable, or SELinux | `grep PROGRAM /etc/mdadm.conf`; `ls -l /usr/local/sbin/disk-alert.sh`; `sudo ausearch -m avc -ts recent` |

## 9. Dependencies

- `dnf` needs `/srv/bulk`, which needs `hdd-vg`, which needs array `hdd`.
- The `isos` pool needs `/var/lib/libvirt/isos` mounted.
- Every VM disk needs `ssd-vg/vmpool`, which needs array `pv00`.
- The host itself needs only `boot_efi`, `boot` and `pv00`.
