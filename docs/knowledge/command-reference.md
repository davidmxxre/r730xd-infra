# Command reference (cumulative)

Task-to-command cheat sheet, organised by tool. Each module adds to it.
Started in Module 0.2. Explanations are in the module that introduced the
tool; this file is for recall.

Placeholders are `<UPPER_CASE>`. Anything that writes needs `sudo`.

## Shell patterns

| Task | Command |
|---|---|
| Write a root-owned file | `echo text \| sudo tee FILE` |
| Append to a root-owned file | `echo text \| sudo tee -a FILE` |
| Write a multi-line root-owned file | `sudo tee FILE >/dev/null <<'EOF'` ... `EOF` |
| Repeat the last command with sudo | `sudo !!` |
| Use a command's output in another | `$(command)` |
| Loop over a list | `for x in a b c; do ...; done` |
| Loop over lines of output | `command \| while read -r line; do ...; done` |
| Real device behind a symlink | `readlink -f <PATH>` |
| Keep a copy before editing | `sudo cp -a FILE FILE.pre-<TAG>` |

## Hardware and block devices

| Task | Command |
|---|---|
| PCI devices with drivers | `lspci -nnk` |
| One PCI class (RAID = 0104, Ethernet = 0200) | `lspci -nnk -d ::<CLASS>` |
| Disks with serial and model | `lsblk -d -o NAME,SERIAL,SIZE,MODEL,ROTA` |
| Whole storage tree | `lsblk` |
| Filesystems, labels, UUIDs, mounts | `lsblk -f` |
| TRIM support | `lsblk -D` |
| Stable disk names | `ls /dev/disk/by-id/ \| grep -v part` |
| UUID and type of one device | `sudo blkid <DEVICE>` |
| Signatures on a device (list only) | `sudo wipefs <DEVICE>` |
| Erase all signatures (destructive) | `sudo wipefs -a <DEVICE>` |
| Loaded kernel modules | `lsmod \| grep <NAME>` |
| Secure Boot state | `mokutil --sb-state` |

## md RAID (`mdadm`)

| Task | Command |
|---|---|
| Quick status of all arrays | `cat /proc/mdstat` |
| Full state of one array | `sudo mdadm --detail /dev/md/<NAME>` |
| Superblock of one member | `sudo mdadm --examine <MEMBER>` |
| Create RAID5 with PPL | `sudo mdadm --create /dev/md/<NAME> --name=<NAME> --level=5 --raid-devices=3 --chunk=512 --consistency-policy=ppl <DISK1> <DISK2> <DISK3>` |
| Wait for a rebuild | `sudo mdadm --wait /dev/md/<NAME>` |
| `ARRAY` lines for the config | `sudo mdadm --detail --scan` |
| Arrays recorded in the config | `grep '^ARRAY' /etc/mdadm.conf` |
| Arrays assembled from the kernel command line | `cat /proc/cmdline` (look for `rd.md.uuid=`) |
| File inside the initramfs | `lsinitrd -f etc/<FILE>` |
| Rebuild the initramfs (running kernel) | `sudo dracut -f` |
| Fail and remove a member | `sudo mdadm /dev/md/<NAME> --fail <MEMBER> --remove <MEMBER>` |
| Add a replacement | `sudo mdadm /dev/md/<NAME> --add <MEMBER>` |
| Re-add an ejected healthy member | `sudo mdadm /dev/md/<NAME> --re-add <MEMBER>` |
| Remove bitmap / add bitmap | `sudo mdadm --grow /dev/md/<NAME> --bitmap=none` / `--bitmap=internal` |
| Enable / disable PPL | `sudo mdadm --grow /dev/md/<NAME> --consistency-policy=ppl` / `=resync` |
| Send a test alert | `sudo mdadm --monitor --scan --test --oneshot` |
| Start a scrub | `echo check \| sudo tee /sys/block/<MDN>/md/sync_action` |
| Stop a scrub | `echo idle \| sudo tee /sys/block/<MDN>/md/sync_action` |
| Scrub result | `cat /sys/block/<MDN>/md/mismatch_cnt` |
| Kernel name of an array | `basename "$(readlink -f /dev/md/<NAME>)"` |
| Stop an array | `sudo mdadm --stop /dev/md/<NAME>` |
| Erase a member's superblock (destructive) | `sudo mdadm --zero-superblock <MEMBER>` |

## LVM

| Task | Command |
|---|---|
| Summaries | `sudo pvs`, `sudo vgs`, `sudo lvs` |
| Include hidden thin-pool parts | `sudo lvs -a` |
| Thin pool usage | `sudo lvs -o lv_name,lv_size,data_percent,metadata_percent <VG>` |
| Make a PV, then a VG | `sudo pvcreate <DEVICE>`; `sudo vgcreate <VG> <DEVICE>` |
| Thick LV | `sudo lvcreate -L <SIZE> -n <LV> <VG>` |
| Thin pool | `sudo lvcreate --type thin-pool -n <POOL> -l 90%FREE --chunksize 512K <VG>` |
| Thin LV | `sudo lvcreate --type thin --thinpool <POOL> -V <SIZE> -n <LV> <VG>` |
| Snapshot of a thin LV | `sudo lvcreate -s -n <SNAP> <VG>/<LV>` |
| Snapshot of a thick LV | `sudo lvcreate -s -L <SIZE> -n <SNAP> <VG>/<LV>` |
| Roll back to a snapshot | `sudo lvconvert --merge <VG>/<SNAP>` |
| Grow an LV and its filesystem | `sudo lvextend -r -L +<SIZE> <VG>/<LV>` |
| Remove an LV | `sudo lvremove <VG>/<LV>` |
| Turn off pool zeroing | `sudo lvchange -Zn <VG>/<POOL>` |

## Filesystems and mounts

| Task | Command |
|---|---|
| Make an XFS filesystem (destructive) | `sudo mkfs.xfs <DEVICE>` |
| Mount read-only | `sudo mount -o ro <DEVICE> <DIR>` |
| Mount everything in fstab | `sudo mount -a` |
| Unmount | `sudo umount <DIR>` |
| What is mounted at a path (one path) | `findmnt <PATH>` |
| Which filesystem paths are on (many paths) | `df -hT <PATH> <PATH>` |
| Check `/etc/fstab` for mistakes | `sudo findmnt --verify` |
| Make systemd re-read fstab | `sudo systemctl daemon-reload` |
| Copy a tree preserving attributes | `sudo cp -a <SRC> <DESTDIR>/` |

## Packages (`dnf`, `rpm`)

| Task | Command |
|---|---|
| Enabled repositories | `dnf repolist` (`-v` for package counts) |
| Drop cached metadata | `sudo dnf clean all` |
| Install / remove | `sudo dnf install <PKG>` / `sudo dnf remove <PKG>` |
| Install a group | `sudo dnf group install "<GROUP>"` |
| List groups | `dnf group list` |
| Which package ships a file | `dnf provides '*/<FILE>'` |
| Is a package installed | `rpm -q <PKG>` |
| Files in a package | `rpm -ql <PKG>` |
| Which package owns a file | `rpm -qf <PATH>` |
| Past transactions | `dnf history` |

## Services and logs (`systemctl`, `journalctl`)

| Task | Command |
|---|---|
| State and recent log | `systemctl status <UNIT>` |
| Start at boot and now | `sudo systemctl enable --now <UNIT>` |
| Stop now and at boot | `sudo systemctl disable --now <UNIT>` |
| Restart | `sudo systemctl restart <UNIT>` |
| One-word checks | `systemctl is-active <UNIT>`; `systemctl is-enabled <UNIT>` |
| Failed units | `systemctl --failed` |
| Timers and next run | `systemctl list-timers` |
| Log of one unit, this boot | `journalctl -u <UNIT> -b` |
| Log by tag | `journalctl -t <TAG>` |
| Kernel messages | `journalctl -k -b` |
| Recent window | `journalctl --since "-10 min"` |
| Follow | `journalctl -f` |
| Write a log line | `logger -t <TAG> "message"` |

## udev (`udevadm`)

| Task | Command |
|---|---|
| Attributes a rule can match | `udevadm info -a -n <DEVICE>` |
| Reload rules | `sudo udevadm control --reload` |
| Re-apply rules to block devices | `sudo udevadm trigger --action=change --subsystem-match=block` |
| Dry-run rules for one device | `sudo udevadm test /sys/block/<SDX>` |
| A disk's SCSI timeout | `cat /sys/block/<SDX>/device/timeout` |

## SMART (`smartctl`)

All disks on r730xd-01 take `-d sat`.

| Task | Command |
|---|---|
| How disks are addressed | `sudo smartctl --scan-open` |
| Health verdict | `sudo smartctl -H -d sat <DISK>` |
| Attributes | `sudo smartctl -A -d sat <DISK>` |
| Everything | `sudo smartctl -a -d sat <DISK>` |
| Start a long self-test | `sudo smartctl -t long -d sat <DISK>` |
| Self-test results | `sudo smartctl -l selftest -d sat <DISK>` |
| Error-recovery control support | `sudo smartctl -l scterc -d sat <DISK>` |

## Virtualization (`virsh`, libvirt)

| Task | Command |
|---|---|
| Which instance am I talking to | `virsh uri` |
| Host readiness | `virt-host-validate qemu` |
| All VMs | `virsh list --all` |
| Start / clean shutdown / power off | `virsh start <VM>` / `virsh shutdown <VM>` / `virsh destroy <VM>` |
| VM's disks and NICs | `virsh domblklist <VM>`; `virsh domiflist <VM>` |
| VM definition | `virsh dumpxml <VM>`; `virsh edit <VM>` |
| Serial console of a VM | `virsh console <VM>` |
| Start at boot | `virsh autostart <VM>` |
| Pools | `virsh pool-list --all`; `virsh pool-info <POOL>` |
| Define a directory pool | `virsh pool-define-as <POOL> dir --target <DIR>` |
| Start a pool now and at boot | `virsh pool-start <POOL>`; `virsh pool-autostart <POOL>` |
| Re-scan a pool | `virsh pool-refresh <POOL>` |
| Files in a pool | `virsh vol-list <POOL>` |
| Stop and forget a pool (data kept) | `virsh pool-destroy <POOL>`; `virsh pool-undefine <POOL>` |
