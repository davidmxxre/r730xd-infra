# Postmortem: iDRAC unresponsive on the management link

- **Date:** 2026-10-23
- **Duration:** about 30 minutes of active troubleshooting (≈ 19:45–20:16 CDT); start
  of the fault unknown (the host had been off since ≈ 04:00 CDT)
- **Impact:** no out-of-band access (SSH, web UI) to the R730xd. No data at risk;
  the host was powered off.
- **Severity:** low (lab, operator on site)
- **Format:** blameless

## Summary

At the start of a work session the iDRAC did not answer ping or SSH from the admin
VM. Testing from a second machine (the Mac, directly on the same cable) showed the
iDRAC itself was not answering ARP even though link LEDs were green. An AC power
cycle restored it. The admin VM then still failed because VMware Fusion's bridge
had gone stale; reconnecting the virtual adapter fixed that.

## Timeline (CDT)

| Time | Event |
| --- | --- |
| ≈ 04:00 | Host shut down cleanly at the end of the previous session; iDRAC on standby power |
| ≈ 19:45 | VM: SSH to the iDRAC fails; `ip neigh` shows `FAILED` after a ping |
| 19:51 | VM interface and Fusion bridge settings confirmed correct |
| 19:56 | Mac with manual 192.168.0.99/24 also gets no reply; link LEDs green on both ends |
| ≈ 20:00 | Both PSUs unplugged for 30 s (DIMM swap done in the same window) and reconnected |
| ≈ 20:16 | Mac pings the iDRAC; VM still fails |
| ≈ 20:26 | Fusion adapter disconnected/reconnected, ARP flushed, profile re-applied; VM connected |

## Root cause

The iDRAC stopped responding to ARP while the host was off. A hang of this kind
in old iDRAC firmware is the most likely cause; iDRAC8 is end-of-life and
2.86.86.86 receives no further fixes. A separate, contributing fault: the Fusion
bridge on the Mac had gone stale (likely after the Mac slept), which blocked the
VM even after the iDRAC recovered.

## Detection

Manual: the operator could not connect. No monitoring exists yet (Phase 1).

## What went well

- Testing from the middle of the path (the Mac) isolated the fault in one step.
- Ignoring the VM's `ethtool` "link detected" (always true for a virtual NIC)
  avoided a false lead.

## What went poorly

- The standby thermal baseline taken afterwards was less settled because the
  AC cycle reset the chassis state.
- No way to tell when the hang began.

## Action items

| Action | Owner | Due |
| --- | --- | --- |
| Add "iDRAC unreachable" procedure to the iDRAC runbook (done) | Dave | 2026-10-25 |
| Phase 1 monitoring: alert when the iDRAC stops answering on TCP 22/443, from a host not on this server | Dave | Phase 1.5 |
| Note `racadm racreset soft` as the first step when SSH works but web UI doesn't (done) | Dave | 2026-10-25 |
| If hangs recur, record frequency and consider a scheduled `racreset` | Dave | ongoing |
