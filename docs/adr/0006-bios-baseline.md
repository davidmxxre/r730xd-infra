# ADR-0006: BIOS baseline

- **Status:** Accepted
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** none

## Context

The host is a headless KVM hypervisor managed over SSH and the iDRAC. It needs
UEFI boot, virtualization extensions, a serial console reachable through the
iDRAC, and predictable behavior after a power cut.

## Decision

| Attribute | Value | Reason |
| --- | --- | --- |
| `BIOS.BiosBootSettings.BootMode` | `Uefi` | Required for Secure Boot and GPT boot |
| `BIOS.ProcSettings.ProcVirtualization` | `Enabled` | VT-x/VT-d for KVM and device passthrough |
| `BIOS.SysProfileSettings.SysProfile` | `PerfPerWattOptimizedOs` | OS (`tuned`) controls CPU power states; lower idle draw |
| `BIOS.SysSecurity.AcPwrRcvry` | `Last` | Return to previous power state after an outage; a deliberate shutdown stays off |
| `BIOS.SerialCommSettings.SerialComm` | `OnConRedirCom2` | BIOS console redirected to COM2 |
| `BIOS.SerialCommSettings.SerialPortAddress` | **`Serial1Com1Serial2Com2`** | Puts the **iDRAC virtual serial port** (serial device 2) on COM2 |
| `BIOS.SerialCommSettings.FailSafeBaud` | `115200` | Matches the Linux console setting |
| `BIOS.SerialCommSettings.ConTermType` | `Vt100Vt220` | Terminal type |
| `BIOS.SerialCommSettings.RedirAfterBoot` | `Enabled` | Keeps redirection through the GRUB menu |
| `BIOS.SysSecurity.SecureBoot` | `Enabled` | Only signed boot code runs (applied as a separate job) |
| `iDRAC.SerialRedirection.Enable` | `Enabled` | iDRAC side of the serial console |

Linux side (host OS module): `console=tty0 console=ttyS1,115200n8` on every
kernel via `grubby --update-kernel=ALL`.

Applied by Lifecycle Controller jobs: `JID_927535322557` (baseline),
`JID_927556587727` (Secure Boot), `JID_927615716627` (serial port address fix).

## History of the serial port fix

The first baseline set `SerialComm=OnConRedirCom2` but left `SerialPortAddress`
at `Serial1Com2Serial2Com1`, which put the iDRAC on COM1 and the physical
connector on COM2. Every value looked right, but `console com2` showed a black
screen. Fixed by setting `Serial1Com1Serial2Com2`. Rule: **test the serial path at
POST before installing an OS.**

## Secure Boot caveat

Microsoft's 2011 UEFI CA, which signs current shims, expires in 2026. Future RHEL
shims signed only by the 2023 CA may not boot on this end-of-life BIOS. Before any
shim update, check the signing CA. Recovery if it fails to boot:
`racadm set BIOS.SysSecurity.SecureBoot Disabled` plus a BIOS job.

## Consequences

- BIOS changes need a Lifecycle Controller job and a reboot. With an OS installed,
  create the job **without** `-r pwrcycle` and reboot cleanly from the OS.
- No BIOS setup password is set (see ADR-0004 residuals).
