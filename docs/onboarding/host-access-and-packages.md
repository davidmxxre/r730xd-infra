# Onboarding: reaching r730xd-01 and working on it

State after Module 0.2. Written for someone who has not touched the host
for a month.

## 1. What the host is and is not connected to

- It is **not** on the home LAN and has no internet access. That is
  deliberate until Module 0.5.
- The only way in is the iDRAC, on a direct cable to the admin VM
  (provisional, ADR-0002), and through it the serial console.
- NTP is off. The clock was set by hand; treat timestamps as approximate.

## 2. Getting a shell

From the admin VM:

```
ssh idrac
console com2
```

`ssh idrac` uses the host alias set up in Module 0.1 (iDRAC address
192.168.0.120). `console com2` attaches to the server's serial console.
Log in as `dave`. Root is locked; use `sudo`.

First thing in every session, because the serial console mangles pasted
text otherwise:

```
bind 'set enable-bracketed-paste off'
```

Then confirm you are where you think you are:

```
hostname
virsh uri
cat /proc/mdstat
```

Expect `r730xd-01`, `qemu:///system`, and four healthy arrays.

## 3. Your account

| | |
|---|---|
| User | `dave` |
| Groups that matter | `wheel` (sudo), `libvirt` (manage the system hypervisor without sudo) |
| Set in `~/.bashrc` | `export LIBVIRT_DEFAULT_URI=qemu:///system` |

If `virsh uri` prints `qemu:///session`, run `source ~/.bashrc`. Anything
defined in the session instance is invisible to the real hypervisor.

## 4. Rules that prevent the mistakes already made once

1. Disk letters (`sda`, `sdb`, ...) move. Use `/dev/disk/by-id/...` for
   disks, `/dev/md/<name>` for arrays, `/dev/<vg>/<lv>` for volumes.
2. `sudo` does not apply to `>`. To write a root-owned file:
   `echo text | sudo tee FILE` (add `-a` to append).
3. A shell variable set as dave is gone after `sudo -i`.
4. A broken udev rule fails silently. Read the value back.
5. `tee -a` appends every time you run it. Check the file before running
   an append twice.
6. Type file names carefully when appending to config files. A typo
   creates a new file that nothing reads.

## 5. Where things are

| Thing | Location |
|---|---|
| VM disks | thin LVs in `ssd-vg/vmpool`, seen as `/dev/ssd-vg/<name>` |
| ISO images | `/var/lib/libvirt/isos` (libvirt pool `isos`) |
| Package repository | `/srv/bulk/repo/rhel9.5/{BaseOS,AppStream}` |
| Backup space (empty until 1.6) | `/srv/backups` |
| Disk alert script | `/usr/local/sbin/disk-alert.sh` (logs with tag `disk-alert`) |
| Array definitions | `/etc/mdadm.conf` |
| SMART monitoring config | `/etc/smartmontools/smartd.conf` |
| Seagate timeout rule | `/etc/udev/rules.d/60-raid-disk-timeout.rules` |
| Repo definition | `/etc/yum.repos.d/rhel-dvd.repo` |
| Backups of files edited in 0.2 | `/etc/fstab.pre-0.2`, `/etc/mdadm.conf.pre-0.2`, `/etc/smartmontools/smartd.conf.pre-0.2`, `/boot/initramfs-*.img.pre-mdadm` (if rebuilt) |

Full map: `docs/architecture/storage-layout.md`.

## 6. Installing software while offline

```
findmnt /srv/bulk
sudo dnf install <PACKAGE>
```

`findmnt` must show `/dev/mapper/hdd--vg-bulk`. "Not registered" notices
from `dnf` are normal. Only what shipped on the RHEL 9.5 DVD is available.

## 7. Getting an ISO onto the host while offline

Copy the ISO to a USB stick formatted with a filesystem the host can read,
plug it into the server, then:

```
lsblk -f
sudo mkdir -p /mnt/usb
sudo mount -o ro <USB_PARTITION> /mnt/usb
sudo cp /mnt/usb/<ISO_FILE> /var/lib/libvirt/isos/
sudo umount /mnt/usb
virsh pool-refresh isos
virsh vol-list isos
```

`<USB_PARTITION>` is the new entry that appears in `lsblk -f`.
Inserting a stick can shift the `sdX` letters of the other disks for that
boot.

## 8. Where to go next

- Daily health check and every operational procedure:
  `docs/runbooks/host-storage-virt.md`.
- Why it is built this way: ADRs 0008 and 0011-0015.
- Open risks: runbook sections 6 and 7; ADR-0007.
