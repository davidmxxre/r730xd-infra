# Runbook: R730xd hardware (r730xd-01)

Physical maintenance: disks, memory, PSUs, fans, diagnostics. Inventory and bay
map: `/docs/architecture/hardware/`. iDRAC commands: `/docs/runbooks/idrac.md`.

**Before opening the chassis:** shut down cleanly (`sudo systemctl poweroff`),
unplug **both** PSU cords, wait 30 s, use ESD precautions. Every chassis open is
logged in the SEL ("The chassis is open while the power is off"); that is expected.

---

## 1. Identify a disk (never by `sdX`, never by size)

Neither the locate LED nor the activity LED works for identification in HBA mode
on this server. Identify by serial.

Bay numbering, facing the front: **down each column, left to right**: column 1 is
bays 0–2 (top to bottom), column 2 is 3–5, column 3 is 6–8, column 4 is 9–11. The
iDRAC bay number equals the physical slot.

1. Find the failing disk's serial on the host:
   `lsblk -o NAME,SIZE,MODEL,SERIAL` or `ls -l /dev/disk/by-id/ | grep -v part`.
2. Look the serial up in `/docs/architecture/hardware/disk-bay-map.md` for the
   bay FQDD and physical slot.
3. Cross-check from the iDRAC:
   `ssh idrac racadm storage get pdisks -o | grep -E '^Disk\.Bay|SerialNumber'`.
4. Check the caddy label (bay number plus last four characters of the serial).

Size is never identification: the 870 EVO and the Barracudas are all 931.5 GiB.

**Never pull a disk from a degraded array on a guess.** Pulling the wrong member
of a degraded RAID5 destroys the array.

## 2. Replace a disk in an array

The mdadm steps belong to the storage runbook (storage module). Hardware sequence:

1. In the OS, mark the member failed and remove it from its array(s):
   the SSDs carry partitions in `md127` and (bays 1–2) `md125`/`md126`.
2. Identify the physical slot (section 1).
3. Hot-pull the caddy (the R730xd backplane supports hot swap).
4. Mount the new drive with its connector **flush with the caddy's back edge**;
   2.5" drives go in the 2.5"→3.5" adapter.
5. Insert and latch. Confirm the iDRAC sees it:
   `ssh idrac racadm lclog view -c storage -n 5` shows PDR8 (inserted), and
   `storage get pdisks -o` lists the new serial with `State = Non-Raid`.
6. Partition and add it back to the array(s) per the storage runbook.
7. Update the disk bay map.

## 3. Disk not detected after insertion

1. `ssh idrac racadm storage get controllers -o` → controller present,
   `CurrentControllerMode = HBA`.
2. `ssh idrac racadm storage get enclosures -o` → backplane `BP13G+EXP` present.
3. No caddy LED and no PDR8 event → physical: drive not reaching the backplane.
   Check the drive sits flush at the back of the caddy, the adapter positions a
   2.5" drive correctly, and the caddy is a 13G tray. (This was the cause on
   2026-10-22.)
4. PDR87 resets then PDR3 "not functioning" → marginal connection or failing
   drive. Reseat once; if it repeats, move it to another bay. Fault follows the
   drive → replace the drive; fault stays with the bay → backplane/slot.

## 4. Memory

Slot rules: each CPU has four channels; slots 1/5/9 = channel 0, 2/6/10 =
channel 1, 3/7/11 = channel 2, 4/8/12 = channel 3. Fill slots 1–4 before 5–8,
same on both CPUs. Match part numbers.

Current state: 12 DIMMs, 192 GB: slots 1–3 and 5–7 on both CPUs; slots 4 and 8
empty on both (ADR-0010). One known-good spare in stock.

**Replacing or adding DIMMs**

1. Shut down, unplug, ESD precautions.
2. Seat each DIMM firmly until **both** latches click (the A6 errors came from
   incomplete seating).
3. Power on and **wait for POST to finish** (watch `console com2` for the login
   prompt). Inventory read during POST is stale.
4. Verify:
   ```
   ssh idrac racadm get BIOS.MemSettings.SysMemSize
   ssh idrac racadm hwinventory | grep -c 'InstanceID: DIMM'
   ssh idrac racadm getsel | tail -20
   ```
   `grep -c` counts DIMM entries. Expect the total you installed and no new
   memory errors.
5. Run the memory test (section 6).

**Diagnosing a missing DIMM, or a DIMM logging errors:** swap it with a
known-good slot on the other CPU (for example A2↔B2). Clear the SEL, boot, wait
30 minutes, read the SEL. If the missing slot or the error follows the stick to
the new slot, the stick is bad; if it stays, the slot or channel is the problem.

**"Correctable memory error rate exceeded for DIMM_xx"** means ECC is still
correcting every error, but so many occurred that the BIOS flagged the module as
degrading. Treat it as a failing stick: swap-test, then remove it. While the
entry is in the SEL, ePSA will not run a memory test.

**The iDRAC shows inventory and SEL from the last boot.** With the host off,
`hwinventory` and memory size reflect the configuration before any DIMM moves.
Power on and let POST finish before reading. After a DIMM change the BIOS stops
at POST with a "memory configuration changed" prompt; acknowledge it (Virtual
Console) once.

**Multi-bit (uncorrectable) memory errors** in the SEL mean the data in memory
may be corrupt. Stop trusting the host until the cause is found and a memory
test passes.

## 5. PSUs

- Two Liteon 750 W (PN W8R3C), hot-swappable. Redundancy policy Input Power
  Redundant; hot spare on (PSU2 sleeps at low load).
- Check: `ssh idrac racadm getsensorinfo | grep -iE 'PS[12] Status|redundan'`.
- Replace a failed PSU with the host running: pull the failed unit, insert a
  matching part number, confirm `Full Redundant`.
- With the host off, PSU status may read `Absent`; check again with it on.

## 6. Hardware diagnostics and memory test

1. **Clear the SEL first.** If it holds any ECC error, ePSA stops the memory test
   with error 2000-0125. Read the newest entries for new memory errors (a new
   one is a real fault: stop), archive the log, clear it (iDRAC runbook §4).
2. Open the Virtual Console (iDRAC runbook section 2) or use a monitor.
3. Power on (host off: web UI → Overview → Server → Power ON/OFF), or reboot
   cleanly from the OS. Press **F10** at POST → Lifecycle Controller.
4. Hardware Diagnostics → Run Hardware Diagnostics. The quick test starts; press
   **Esc** to stop it.
5. Click the right arrow to the advanced view. In the left-hand device list,
   scroll down and **click Memory** (selection is by clicking a name; the green
   checks are earlier results). Tick **Thorough Test Mode**, then **Run Tests**.
   The status bar must change from "Done" to "Testing". On 192 GB the
   Thorough memory test took about one hour (2026-11-03).
6. Record the result and any error code in the module log.

For stronger proof, run MemTest86 overnight for several passes (USB stick, or an
ISO via iDRAC virtual media).

## 7. Fans and thermal

- All fans are redundant; check with `getsensorinfo | grep -iE 'RPM|redundan'`.
- Compare readings with `/docs/architecture/hardware/power-thermal-baseline.md`.
  Rising temperature or fan speed at the same load points to a failing fan,
  blocked intake or missing blanks.
- Idle baseline is already high (≈ 8,800 RPM, 45%); not yet investigated. Start
  with `ssh idrac racadm get System.ThermalSettings`.
- Inlet range: 10–35 °C.

## 8. Known history

| Date | Event | Resolution |
| --- | --- | --- |
| 2026-05-26 | Earlier controller's cache battery failed (BAT1029) | Controller since replaced by the H330 (no battery) |
| 2026-10-22 | Multi-bit memory errors on A2, A4, A6 | Reseated; A4/A8 sticks bad (ADR-0009) |
| 2026-10-22 | No disks detected | Caddy installation corrected |
| 2026-10-23 | Bay 2 SSD dropped (PDR87, PDR3, PDR5) | Reseated; marked suspect |
| 2026-10-23 | Spare DIMMs failed in B4/B8 | Ran at 224 GB (ADR-0009) |
| 2026-10-26 | Correctable-error-rate threshold, DIMM_A2; followed the stick to B2 | Stick removed; 12 DIMMs, 192 GB (ADR-0010) |
| 2026-11-03 | ePSA Thorough memory test | All tests passed |
| 2026-10-23 | iDRAC hung | AC power cycle; postmortem filed |
