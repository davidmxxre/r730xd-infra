# Power and thermal baseline — r730xd-01

Use these numbers to spot degradation later (a failing fan or clogged intake
shows as rising temperature or fan speed at the same load) and to size a UPS.

| State | Date | Room | Inlet | Exhaust | Power | Fans | Source |
| --- | --- | --- | --- | --- | --- | --- | --- |
| No OS, host on (reference only) | 2026-10-22 | — | 22 °C | 32 °C | 196 W | — | `before-getsensorinfo.txt` |
| Standby, host off | 2026-10-23 | 21 °C (70 °F) | 23 °C | 28 °C (not settled) | ≈ 26 W | — | `baseline-standby.txt`; power from `System.Power` `Min.LastWeek` |
| OS idle (load ≈ 0.2) | 2026-10-23 | 21 °C | 23 °C | 28 °C | 124 W | 8,760–8,880 RPM, 45% | `baseline-os-idle.txt`; `System.Power` `Realtime.Power` |
| Peak observed (POST/boot) | 2026-10-23 | — | — | — | 327 W | — | `System.Power` `Max.Power` |
| Daily average | 2026-10-23 | — | — | — | 107 W | — | `System.Power` `Avg.LastDay` |
| OS under load | _to do_ | | | | | | Host OS / storage module |

## Notes

- With the host off, live power and PSU status read `NA` / `Absent`; the iDRAC's
  power history still records standby draw.
- The inlet sensor reads about 2 °C above room temperature (it sits behind the
  drives). Dell's allowed inlet range is 10–35 °C.
- Idle fan speed is high for an R730xd (typically 3,000–4,500 RPM). All sensors
  OK; cause not investigated. Candidates: thermal profile or fan offset,
  third-party PCIe card response, missing bay or slot blanks.

## Commands

```
ssh idrac racadm getsensorinfo > evidence/baseline-<state>.txt
grep -iE 'Inlet|Exhaust|Pwr Consumption|RPM|PS[12] Status' evidence/baseline-<state>.txt
ssh idrac racadm get System.Power
```

- `getsensorinfo` dumps every sensor; save it per state.
- `grep -iE` keeps temperatures, power, fans and PSU status (case-insensitive,
  `|` means "or").
- `get System.Power` shows real-time draw plus averages and min/max per hour,
  day and week, with timestamps.
