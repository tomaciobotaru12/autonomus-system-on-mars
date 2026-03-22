# Scenario 2 — Regional Storm (High Risk, Safe Landing Available)

## Description

A regional storm is detected 12 minutes ahead. It is growing in density and
moving toward the aircraft. Solar panels are already showing degradation.
A pre-mapped landing zone exists 6 minutes away. This scenario tests whether
the system correctly escalates from REROUTE to LAND_SAFE as conditions worsen.

---

## Simulated Sensor State at Detection (T=0)

```
radar:
    density:          0.55 above baseline   (heavy dust)
    distance:         2.6 km
    movement_vector:  moving toward aircraft at ~5 km/h
    growth_rate:      +0.04 per minute      (actively intensifying)
    confidence:       75%
    status:           NOMINAL

optical:
    visibility:       significantly reduced
    confidence:       55%
    status:           DEGRADED              (dust beginning to coat lens)

power_monitor:
    solar_efficiency: 81% of baseline
    battery_level:    65%
    degradation_rate: 1.2% per minute       (actively degrading)
    status:           NOMINAL

imu:
    status:           NOMINAL

barometric:
    pressure_delta:   dropping steadily
    status:           NOMINAL
```

---

## Expected Risk Score Calculation at T=0

```
storm_density_score:    18   (0.55 above baseline = heavy dust)
power_vulnerability:    12   (81% efficiency, moderate degradation rate)
time_pressure_score:    10   (closing distance + storm moving toward aircraft)
uncertainty_penalty:    13   (optical DEGRADED +10, confidence 75% = +3)

TOTAL RISK SCORE:       53
risk_delta:             INCREASING (growth_rate positive, efficiency dropping)
adjusted_risk:          58   (+5 for INCREASING delta)
```

---

## Expected Decision Engine Output at T=0

```
Layer 1 hard rules:     none triggered
adjusted_risk:          58

Layer 2 threshold:      50–75 range
nearest_zone:           EXISTS, 6 minutes away, flatness_rating = 8
Expected action:        LAND_SAFE

Reason: Risk in critical range and actively increasing.
        Safe landing zone reachable within current window.
        Rerouting not viable — storm is closing faster than reroute path allows.
```

---

## How Scenario Evolves if LAND_SAFE is NOT Executed (T+4 minutes)

```
optical.status:         FAILED              (fully coated)
radar.density:          0.72 above baseline
power_monitor.solar:    71% of baseline
battery_level:          60%
risk_score:             74
adjusted_risk:          84  (optical FAILED adds to uncertainty_penalty)

Layer 1 hard rules:     none yet
Layer 2:                LAND_IMMEDIATE      (adjusted_risk > 75)

Landing zone from T=0:  now only 2 minutes away but barely reachable
```

This branch shows why acting at T=0 (LAND_SAFE) is safer than waiting.
At T+4, the system is forced into LAND_IMMEDIATE with a closing window.

---

## What This Scenario Tests

- System correctly scores heavy + growing storm as high risk
- LAND_SAFE triggered appropriately when zone is reachable
- Escalation to LAND_IMMEDIATE demonstrated if LAND_SAFE is delayed
- Optical DEGRADED status correctly raises uncertainty_penalty
- risk_delta INCREASING correctly adds to adjusted_risk

---

## Pass Condition

Action = LAND_SAFE at T=0.
Aircraft navigates to pre-mapped zone before optical fully fails.
If LAND_SAFE is delayed: action = LAND_IMMEDIATE at T+4. Both are acceptable.
Action = CONTINUE at any point in this scenario = FAIL.
