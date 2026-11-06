# ADR-0010: Run at 192 GB, 12 DIMMs, symmetric across both CPUs

- **Status:** Accepted
- **Date:** 2026-11-03
- **Module:** 0.1
- **Supersedes:** ADR-0009

## Context

ADR-0009 ran the host at 14 DIMMs (224 GB) with two failing sticks pulled from
B4/B8. After it was written:

- 2026-10-26: within six minutes of a cleared SEL, the BIOS logged
  **"Correctable memory error rate exceeded for DIMM_A2"**. ePSA refused to start a
  memory test while the log held ECC records (error 2000-0125, then 2000-0251).
- A swap test (A2↔B2) moved the error to **DIMM_B2**: the stick is bad, not the
  slot or CPU1 channel 1. This is the stick behind the 2026-10-22 multi-bit
  errors logged against A2.
- That made four failed sticks in total from the original 16, plus two failed
  spares.

Removing the bad stick left 13 good sticks and broke the fill order on CPU2
(B6 populated without B2).

## Decision

Run **12 DIMMs, 192 GB, symmetric**: 96 GB per CPU, channels 0–2 with two DIMMs
each, channel 3 empty on both CPUs.

| Slot | CPU1 | CPU2 |
| --- | --- | --- |
| 1, 2, 3 | populated | populated |
| 4 | empty | empty |
| 5, 6, 7 | populated | populated |
| 8 | empty | empty |
| 9–12 | empty | empty |

Moves made: bad stick removed from B2; A4 → B2; A8 → spare stock.

**Validation:** Dell ePSA 4239.45 **Thorough** memory test, 2026-11-03:
**All tests passed.** SEL clear of memory errors after the rebuild.

## Alternatives considered

| Option | Why not |
| --- | --- |
| 13 DIMMs, 208 GB (CPU1 128 GB, CPU2 80 GB) | Unequal NUMA nodes and one CPU2 channel with a single DIMM; 16 GB is not worth the imbalance |
| Keep 14 DIMMs with the bad stick | It logs correctable-error-rate events every boot, blocks diagnostics, and already produced a multi-bit (uncorrectable) error once |

## Consequences

- **Capacity planning uses 192 GB.** About 187 GiB is usable by the OS; leave
  headroom for the host and plan VMs per NUMA node (96 GB each).
- Equal NUMA nodes: a VM sized under ~90 GB can be pinned to one node.
- Each CPU loses its fourth memory channel; bandwidth is lower than a fully
  balanced 4-channel layout. Acceptable for a lab.
- **Spare stock:** one known-good 16 GB stick (from A8, labeled GOOD-SPARE).
  Failed sticks labeled BAD: the original A4 and A8 sticks, the A2 stick, and both
  earlier spares.
- **Restoring 256 GB** needs four sticks matching the working sticks' part number
  (`racadm hwinventory | grep -A30 'InstanceID: DIMM.Socket.A1'`), placed in
  A4, A8, B4, B8.
- ePSA's memory test is not exhaustive. If memory errors reappear in the SEL, run
  MemTest86 overnight for several passes before trusting the host.
