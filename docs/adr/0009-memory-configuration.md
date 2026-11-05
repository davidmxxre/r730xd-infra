# ADR-0009: Run at 224 GB with CPU2 channel 3 empty

- **Status:** **Superseded by ADR-0010** (2026-11-03). The A2 stick was found
  bad; the host now runs 12 DIMMs, 192 GB.
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** none

## Context

The server holds 16 × 16 GB DDR4-2133 RDIMMs (A1–A8, B1–B8), 256 GB nominal.
Planning documents said 336 GB, then 224 GB; the true installed total was 256 GB.

- On 2026-10-22 the SEL logged multi-bit (uncorrectable) memory errors on DIMM_A2,
  A4 and A6, traced to improper seating.
- A4 and A8 were then missing from inventory. A swap test (A4↔B4, A8↔B8) moved the
  missing pair to B4 and B8: **two bad sticks**, not a CPU1 channel fault.
- Two spare sticks placed in B4 and B8 also failed to train.

Four used sticks have failed. They may simply be faulty, or be a different type
(rank, or LRDIMM vs RDIMM) from the twelve that work.

## Decision

- Run with **14 DIMMs, 224 GB** (OS sees about 219 GiB). CPU1 has all four
  channels populated; CPU2 has channel 3 (B4/B8) empty.
- Remove the failing sticks from B4 and B8 rather than leave them mapped out.
- Do not add the mismatched 8 GB sticks on hand.
- Future purchases must match the working sticks' part number
  (`racadm hwinventory | grep -A30 'InstanceID: DIMM.Socket.A1'`).

## Consequences

- CPU2 loses a little memory bandwidth. Acceptable for a lab.
- **A2 and A6 are not proven good** until a thorough memory test of the 14-DIMM
  configuration passes. Nothing stateful should be trusted on this host before
  that test.
- Capacity planning uses 224 GB.
