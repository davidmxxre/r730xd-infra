# Runbook: iDRAC (r730xd-01)

Out-of-band management for the R730xd: power, console, BIOS, hardware health.
Written to be run cold. Access setup is in `/docs/onboarding/idrac-access.md`.

| Fact | Value |
| --- | --- |
| Address | 192.168.0.120/24 (provisional direct cable, ADR-0002) |
| Admin | `dave` (slot 3), SSH key `~/.ssh/id_rsa_idrac`; root disabled (ADR-0003) |
| Firmware | iDRAC8 2.86.86.86, final release, no more security fixes |
| License | Enterprise (Virtual Console available) |
| Open services | TCP 22, 80→443, 443 (TLS 1.2), 5900 (Virtual Console); UDP 623 RMCP ping only (ADR-0004) |
| NTP | 192.168.0.10 (admin VM, provisional) |

All commands run from the admin VM. `ssh idrac racadm <cmd>` runs one RACADM
command and returns its output; `ssh idrac` with no command opens the
interactive RACADM shell.

---

## 1. Health check (start of every session)

```
ssh idrac racadm getsysinfo | grep -E 'Firmware Version|Power Status|RAC Date'
ssh idrac racadm getsensorinfo | grep -viE ' ok |^\[|^Sensor Type|^$'
ssh idrac racadm getsensorinfo | grep -i redundan
ssh idrac racadm getsel | tail -20
ssh idrac racadm getractime; date -u
```

| Command | Healthy result |
| --- | --- |
| `getsysinfo \| grep …` | Firmware 2.86.86.86, power state as expected |
| `getsensorinfo \| grep -viE …` | `-v` inverts: shows only lines that are **not** OK, headers or blank. Expect only header-like lines or nothing alarming |
| `… \| grep -i redundan` | `PS Redundancy: Full Redundant`, `Fan Redundancy: Full Redundant` |
| `getsel \| tail -20` | No new Critical entries since the last check |
| `getractime; date -u` | Within a few seconds (if the VM, the NTP source, has been running) |

---

## 2. Consoles

### Serial console (text, via SSH)

```
ssh idrac
console com2
```

Press Enter to redraw the login prompt. Exit with **Ctrl+\\**. Works from POST
through GRUB to the OS login (`r730xd-01 login:`), because BIOS redirection
(COM2, `SerialPortAddress=Serial1Com1Serial2Com2`) and the kernel
(`console=ttyS1,115200n8`) both point at the iDRAC's virtual serial port.

Black screen? Press Enter first. Then check in order:
1. `ssh idrac racadm get BIOS.SerialCommSettings` → `SerialComm=OnConRedirCom2`,
   `SerialPortAddress=Serial1Com1Serial2Com2`, `RedirAfterBoot=Enabled`.
2. `ssh idrac racadm get iDRAC.SerialRedirection.Enable` → `Enabled`.
3. On the host: `cat /proc/cmdline` contains `console=ttyS1,115200n8`;
   `systemctl status serial-getty@ttyS1` is active.

### Virtual Console (graphical, via browser)

Mac browser → `https://192.168.0.120` → log in as `dave` → Dashboard → Virtual
Console → Launch. Use it for F2 (System Setup), F10 (Lifecycle Controller,
Hardware Diagnostics) and F11 (boot menu). On a Mac keyboard, send F-keys with
Fn, or from the console's Keyboard menu.

---

## 3. Power operations

| Goal | Command | Notes |
| --- | --- | --- |
| Check state | `ssh idrac racadm serveraction powerstatus` | `ON` or `OFF` |
| Clean shutdown | On the host: `sudo systemctl poweroff` | Preferred |
| Clean shutdown, host unreachable | `ssh idrac racadm serveraction graceshutdown` | Sends the OS an ACPI power-button press |
| Clean reboot | On the host: `sudo systemctl reboot` | |
| Power on | `ssh idrac racadm serveraction powerup` | |
| Hard off / hard cycle | `serveraction powerdown` / `powercycle` | **Like pulling the plug.** Only with no OS, or a hung OS |

AC recovery is `Last`: after a power cut the host returns to its previous state.

---

## 4. Reading logs

| Log | Command | Contains |
| --- | --- | --- |
| System Event Log (SEL) | `ssh idrac racadm getsel` | Hardware events: memory errors, chassis open, PSU, drive insert/remove |
| Lifecycle Controller log | `ssh idrac racadm lclog view -n 50` | Detailed events with message IDs |
| LC log, storage only | `ssh idrac racadm lclog view -c storage -n 15` | Disk resets, failures, inserts |

Useful storage message IDs: **PDR8** inserted, **PDR5** removed, **PDR87** disk
reset, **PDR3** disk not functioning (critical).

Clear the SEL only after archiving it:
`ssh idrac racadm getsel > evidence/sel-<date>.txt && ssh idrac racadm clrsel`.

**ePSA refuses to run a memory test while the SEL holds ECC errors** (error
2000-0125: "degraded or disabled ECC functionality"). Before a memory test:
read the newest SEL entries for any *new* memory error, archive the SEL, clear
it, reboot, then run the test. Web UI path: Overview → Server → Logs → System
Event Log → Save As, then Clear Log.

---

## 5. Settings and jobs

- `racadm get <Group>` reads a whole group; `get <Group.Attribute>` one value.
- `racadm help <Group.Attribute>` lists the accepted values. Check before `set`.
- **iDRAC attributes** apply immediately on `set`.
- **BIOS and storage attributes** only stage a pending value. A Lifecycle
  Controller job applies them at reboot. Only one BIOS job can be pending.

Apply a staged BIOS change **with the OS running**:

```
ssh idrac racadm jobqueue create BIOS.Setup.1-1 -s TIME_NOW
sudo systemctl reboot        # on the host
ssh idrac racadm jobqueue view
```

- Omitting `-r` queues the job without forcing a power cycle; the clean OS reboot
  triggers it.
- Use `-r pwrcycle` only when no OS is running.
- Done when the job shows `Status=Completed` and `PR19: Job completed successfully`,
  and `get` shows no `Pending Value`.

Any command that answers "operation initiated" with a job ID runs in the
background. **Wait for `Completed` before using its result.**

Clear a stuck job queue (also drops legitimate pending jobs):
`ssh idrac racadm jobqueue delete -i JID_CLEARALL`.

---

## 6. Expected configuration (verify after any change)

```
for g in iDRAC.IPMILan iDRAC.VNCServer iDRAC.Telnet iDRAC.SNMP iDRAC.WebServer \
  iDRAC.VirtualConsole iDRAC.IPBlocking iDRAC.NTPConfigGroup iDRAC.Time; do
  echo "== $g"; ssh idrac racadm get $g
done
```

| Attribute | Expected |
| --- | --- |
| `iDRAC.IPMILan.Enable` | Disabled |
| `iDRAC.VNCServer.Enable` | Disabled |
| `iDRAC.Telnet.Enable` | Disabled |
| `iDRAC.SNMP.AgentEnable` | Disabled |
| `iDRAC.WebServer.TLSProtocol` | TLS 1.2 Only (value 2) |
| `iDRAC.VirtualConsole.Enable` / `EncryptEnable` | Enabled / Enabled |
| `iDRAC.IPBlocking.BlockEnable` / `FailCount` / `FailWindow` / `PenaltyTime` | Enabled / 5 / 60 / 600 |
| `iDRAC.NTPConfigGroup.NTPEnable` / `NTP1` | Enabled / 192.168.0.10 |
| `iDRAC.Time.Timezone` | UTC |
| `iDRAC.Users.2.Enable` | Disabled |
| `iDRAC.Users.3` | `UserName=dave`, `Privilege=0x1ff`, `Enable=Enabled` |
| `System.Power.RedundancyPolicy` | Input Power Redundant |
| `LifecycleController.LCAttributes.CollectSystemInventoryOnRestart` | Enabled |
| BIOS values | ADR-0006 table |

External check: the nmap/ipmitool commands in ADR-0004.

---

## 7. Configuration export (Server Configuration Profile)

Firmware RACADM can only export to a network share, so the admin VM serves NFS
temporarily. The exports file `/etc/exports.d/idrac.exports` already contains
`/srv/idrac-export 192.168.0.120(rw,sync)`.

```
sudo systemctl start nfs-server
sudo firewall-cmd --add-service={nfs,rpc-bind,mountd}
ssh idrac racadm get -t xml -f r730xd-scp.xml -l 192.168.0.10:/srv/idrac-export
ssh idrac racadm jobqueue view | grep -A4 'Export'     # repeat until Completed
cp /srv/idrac-export/r730xd-scp.xml ~/r730xd/evidence/
grep -iE 'password|community|secret' ~/r730xd/evidence/r730xd-scp.xml | grep -v '\*\*\*\*\*\*'
sudo systemctl stop nfs-server
sudo firewall-cmd --reload
```

- `firewall-cmd --add-service=…` without `--permanent` opens the ports only until
  the next reload, so `--reload` closes them again.
- The `grep … | grep -v` line lists secret-looking lines **without** the `******`
  mask. Expected: only the SNMP/IPMI community names, a policy flag, empty iSCSI
  CHAP fields and `PasswordStatus`. Anything else: do not commit the file.

Re-export after every iDRAC or BIOS change.

---

## 8. Account management

| Task | Command |
| --- | --- |
| Show the account | `ssh idrac racadm get iDRAC.Users.3` |
| Change password (≤ 20 characters, no `'`) | `read -rs NEWPW` (on its own line), then `ssh idrac "racadm set iDRAC.Users.3.Password '$NEWPW'"; unset NEWPW` |
| Add a second key | `ssh idrac "racadm sshpkauth -i 3 -k 2 -t '$(cat <newkey>.pub)'"` |
| Remove a key | `ssh idrac racadm sshpkauth -i 3 -k <slot> -d` |

Never run `read -rs` inside a pasted block: it swallows the next pasted line.

---

## 9. Firmware updates

Baseline and freeze rules: ADR-0005. Procedure:

1. Download from Dell support (service tag 5VWPHK2); check the SHA-256 with
   `sha256sum <file>`.
2. Upload: web UI → iDRAC Settings → Update and Rollback, or remote RACADM:
   `racadm -r 192.168.0.120 -u dave -p "$PW" --nocertwarn update -f <file>`.
3. iDRAC packages apply immediately (the iDRAC restarts; host unaffected).
   BIOS/PERC/NIC/CPLD packages queue a job for the next reboot; reboot cleanly
   from the OS.
4. Verify with `swinventory`; update `/docs/architecture/hardware/firmware-baseline.md`.
5. CPLD updates go alone. Never flash a package for a different part number.

---

## 10. Troubleshooting

### iDRAC unreachable (ping/SSH fail)

Work from the middle of the path:

1. **From the VM:** `ping -c 3 192.168.0.120; ip neigh show dev <bridged-iface>`.
   `FAILED`/`INCOMPLETE` = no layer-2 answer. (Inside a VM `ethtool` always says
   link up; ignore it.)
2. **From the Mac directly** (adapter set to 192.168.0.99/24):
   `ping -c 3 192.168.0.120`.
   - Mac works, VM fails → stale Fusion bridge. Fusion: Virtual Machine → Network
     Adapter → Disconnect, then Connect. In the VM:
     `sudo ip neigh flush dev <iface>; nmcli connection down idrac-link; nmcli connection up idrac-link`.
     Prove the VM reaches the wire with `ping -c 3 192.168.0.99`.
   - Mac fails too, link LEDs green → **the iDRAC has hung.** Go to 3.
3. **Reset the iDRAC.** If SSH still works but the web UI doesn't:
   `ssh idrac racadm racreset soft`. If nothing answers: shut the host down
   cleanly if possible, then unplug **both** PSU cords for 30 s, reconnect, wait
   about 3 minutes. Write a postmortem (see `/docs/incidents/`).
4. **Locked out by IP blocking** (5 failed logins in 60 s from one address): wait
   10 minutes, try once. A different source address (the Mac) is not blocked.

### Lost all admin access

Physical console → F2 at POST → iDRAC Settings → User Configuration → re-enable
or reset an account. Then reinstall the SSH key.

### "Could not resolve hostname idrac"

The `Host idrac` block is missing from `~/.ssh/config` on this machine, or `$IDRAC`
is unset in a new shell. See the onboarding doc.

---

## 11. Known quirks of this iDRAC8

- `racadm hwinventory <FQDD>` gives a syntax error. Filter instead:
  `ssh idrac racadm hwinventory | grep -A30 'InstanceID: <FQDD>'`.
- Locate LEDs (`storage blink`) don't work in HBA mode.
- Inventory read during POST is stale; wait for POST to finish.
- With the host off, live power and PSU sensors read `NA`/`Absent`.
- UDP 623 shows `open` because of RMCP presence pings, even with IPMI off.
