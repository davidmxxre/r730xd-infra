# Hardware inventory — r730xd-01

Source: `racadm getsysinfo`, `hwinventory`, `swinventory`, `license view` on
2026-10-22/23. Raw output is under `evidence/`.

## System

| Item | Value |
| --- | --- |
| Model | Dell PowerEdge R730xd, 12 × 3.5" front bays |
| Service tag | 5VWPHK2 |
| Express service code | 12813300002 |
| Host name | `r730xd-01` |
| Host OS | RHEL 9.5, UEFI, Secure Boot on |
| iDRAC | iDRAC8 with Lifecycle Controller, firmware 2.86.86.86 |
| iDRAC license | iDRAC8 Enterprise (perpetual), plus OpenManage Essentials Advanced Features |
| iDRAC MAC | 18:66:DA:B8:B6:E5 |
| iDRAC address | 192.168.0.120/24 (provisional, ADR-0002) |
| BIOS | 2.19.0 |
| System CPLD | 1.0.1 |

## CPUs

Two sockets (CPU1 owns slots A1–A12, CPU2 owns B1–B12). Model and core count:
see `evidence/before-hwinventory.txt` (`InstanceID: CPU.Socket.1/2`) and fill in.

## Memory (ADR-0010)

| Slots | Contents | State |
| --- | --- | --- |
| A1–A3, A5–A7 | 16 GB DDR4-2133 ECC RDIMM | Detected, OK |
| B1–B3, B5–B7 | 16 GB DDR4-2133 ECC RDIMM | Detected, OK |
| A4, A8, B4, B8 | Empty | Channel 3 unpopulated on both CPUs |
| A9–A12, B9–B12 | Empty | — |

Usable: 12 DIMMs, 192 GB, 96 GB per CPU (NUMA node). Validated by ePSA Thorough
memory test, 2026-11-03: all tests passed.
Channel map: channel 0 = slots 1/5/9, channel 1 = 2/6/10, channel 2 = 3/7/11,
channel 3 = 4/8/12 on each CPU.

Spare stock: 1 × known-good 16 GB stick (GOOD-SPARE). Five sticks labeled BAD.

## Storage

| Item | Value |
| --- | --- |
| Controller | PERC H330 Mini (LSI SAS3008), firmware 25.5.9.0001, **HBA mode**, no cache, no battery |
| Backplane | BP13G+EXP 0:1, 12 slots, firmware 3.35 |
| Chipset SATA | C610/X99 sSATA and 6-port SATA (AHCI), unused |
| Disks | See `disk-bay-map.md` |

## Network

| FQDD | Port | MAC |
| --- | --- | --- |
| NIC.Integrated.1-1-1 | Intel X520 10 GbE SFP+ | E4:43:4B:4C:9C:20 |
| NIC.Integrated.1-2-1 | Intel X520 10 GbE SFP+ | E4:43:4B:4C:9C:22 |
| NIC.Integrated.1-3-1 | Intel I350 1 GbE RJ45 | E4:43:4B:4C:9C:24 |
| NIC.Integrated.1-4-1 | Intel I350 1 GbE RJ45 | E4:43:4B:4C:9C:25 |

rNDC: Intel Ethernet 10G 4P X520/I350, firmware 22.5.7. Port-to-chip mapping
from `swinventory` names; confirm by link LEDs when cabling in the network module.

## Power

| Item | Value |
| --- | --- |
| PSUs | 2 × Liteon 750 W, Dell PN W8R3C (`0W8R3CA02`), firmware 00.23.32 |
| Input | 122–124 V AC |
| Redundancy policy | Input Power Redundant; sensor `Full Redundant` |
| Hot spare | Enabled, primary PSU1 |
| Power cap | Disabled |

## History

- 2026-05-26: battery events from an earlier PERC with cache (the current H330 has
  no battery).
- 2026-10-22: multi-bit memory errors on A2/A4/A6 (reseated); A4/A8 sticks failed.
- 2026-10-23: bay 2 SSD dropped once (PDR3); spare DIMMs failed in B4/B8; iDRAC
  hang (see `/docs/incidents/2026-10-23-idrac-unresponsive.md`).
- 2026-10-26: correctable-error-rate threshold on DIMM_A2; swap test proved the
  stick bad; host rebuilt to 12 DIMMs, 192 GB (ADR-0010).
- 2026-11-03: ePSA Thorough memory test passed on the 12-DIMM layout.
