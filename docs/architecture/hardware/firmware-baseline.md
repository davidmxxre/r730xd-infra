# Firmware baseline — r730xd-01

Checked 2026-10-23 against Dell support for service tag 5VWPHK2 (ADR-0005).
Source: `ssh idrac racadm swinventory` → `evidence/swinventory-now.txt`.

| Component | Installed | Latest found | Action | Notes |
| --- | --- | --- | --- | --- |
| iDRAC8 with Lifecycle Controller | 2.86.86.86 | 2.86.86.86 | None | Final iDRAC8 release; last flashed before Module 0.1 |
| BIOS | 2.19.0 | 2.19.0 | None | |
| PERC H330 Mini | 25.5.9.0001 | 25.5.9.0001 | None | |
| Backplane BP13G+EXP | 3.35 | 3.35 | None | |
| rNDC X520/I350 | 22.5.7 | 22.5.7 | None | |
| System CPLD | 1.0.1 | none listed | None | No CPLD package on Dell support, 2026-10-23 |
| PSU 1 and 2 (Liteon 750 W, PN W8R3C) | 00.23.32 | none listed | None | R730xd page lists no package for W8R3C |
| Dell 64-bit uEFI Diagnostics | 4239A37 | — | — | Utility |
| Dell OS Driver Pack | 16.10.10 | — | — | Utility |
| OS Collector | 6.0 | — | — | Utility |

Verify the live state at any time:

```
ssh idrac racadm swinventory | grep -E 'ElementName|Current Version'
```

`grep -E` keeps each component's name line and its installed-version line.

## How Dell's site was searched

- Drivers & Downloads for the service tag, keyword search per component.
- Firmware is usually listed under a Windows Server OS filter.
- "Chipset" category results are Windows chipset drivers, not firmware.
- PSU firmware is packaged per PSU part number; identify the part first.
