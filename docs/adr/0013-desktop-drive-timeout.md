# ADR-0013: SCSI timeout and scrubbing for the desktop-class disks

- Status: Accepted
- Date: 2026-11-09
- Module: 0.2
- Related: ADR-0001 (controller), ADR-0012

## Context

The `hdd` array is built from Seagate ST1000DM010 desktop disks. Disks sold
for arrays support error-recovery control (ERC): they give up on an
unreadable sector after a few seconds and report an error, so the array
can rebuild that sector from parity. Desktop disks keep retrying, possibly
for longer than the kernel is willing to wait.

The kernel aborts any disk command that exceeds
`/sys/block/<disk>/device/timeout` and resets the device. With the
`megaraid_sas` driver on this host the default is 90 seconds (measured on
all six disks). When a reset happens the md layer sees an I/O error and can
eject a disk that was slow, not dead. During a rebuild that loses the
array.

ERC support on these disks, checked with
`smartctl -l scterc`: ______________________ (paste the output line here).

## Options

| Option | Result |
|---|---|
| Leave the 90 s default | A recoverable bad sector can cost a member |
| Set a short ERC limit on the disk | Correct fix; not available on disks without ERC |
| Raise the kernel timeout above the disk's worst case | The array stalls while the disk retries, but stays whole |
| Replace the disks | Removes the problem; costs money |

## Decision

1. A udev rule sets the timeout to 180 seconds on every disk whose model
   starts with `ST1000DM010`:
   `/etc/udev/rules.d/60-raid-disk-timeout.rules`.
2. `raid-check.timer` is enabled so every array is scrubbed weekly, and a
   scrub can be started by hand at any time.
3. The SSDs keep the driver default.

## Consequences

- A Seagate disk stuck on a bad sector can stall I/O on `hdd` for up to
  three minutes. Acceptable for a bulk and backup tier.
- Latent bad sectors are found weekly while parity can still repair them.
- The kernel timeout is the only layer under our control. The controller
  firmware has its own limits and may still drop a stalled disk.
- udev does not report a malformed rule on the terminal. The value must be
  read back after any change to the rule (runbook validation row 12).

## What would change this decision

Disks with ERC. Then set a short disk-side limit (for example 7 seconds,
`smartctl -l scterc,70,70`) at boot and return the kernel timeout to its
default.
