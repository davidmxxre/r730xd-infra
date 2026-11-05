# ADR-0007: Power protection

- **Status:** **Proposed** — UPS decision open
- **Date:** 2026-10-23
- **Module:** 0.1
- **Supersedes:** none

## Context

Abrupt power loss risks mdadm RAID5 parity inconsistency and LVM thin-pool
metadata damage, on top of downtime. The SSDs are consumer drives (Samsung 850
PRO, 870 EVO) without power-loss protection, so acknowledged writes in their
caches can be lost on a power cut. The H330 has no cache to protect.

Measured on 2026-10-23 (`racadm get System.Power`):

| State | Power |
| --- | --- |
| Standby (host off) | about 26 W |
| OS idle | 124 W |
| Daily average | 107 W |
| Peak observed (POST/boot) | 327 W |

## Decision so far

- **PSU redundancy policy set to Input Power Redundant**
  (`System.Power.RedundancyPolicy=1`). Sensor now reads
  `PS Redundancy: Full Redundant`. Previously `Not Redundant`, which meant no
  "redundancy lost" alert would ever fire.
- **Hot spare kept on** (`HotSpare.Enable=Enabled`, primary PSU1).
- **Power capping off** (`Cap.Enable=Disabled`).
- **UPS: open.** Options:
  1. Buy a line-interactive UPS sized for at least 350 W with runtime for a clean
     shutdown, and configure NUT for automatic shutdown (host OS module).
  2. Accept the risk explicitly and record it here, signed and dated.

## Consequences

- Two PSUs protect against a PSU failure, not a utility outage. Both PSUs are
  likely on one home circuit, so "input power redundant" does not protect
  against losing the circuit.
- With AC recovery = Last, the host returns to its previous state after power
  returns; that does not prevent data damage from the cut itself.
- **This ADR must move to Accepted** (UPS purchased, or risk accepted) before
  the storage module puts data on the arrays that matters.
