# Scenario 1 — Dust Devil (Short-Lived Local Vortex)

## Description

A dust devil is detected 12 minutes ahead. Dust devils on Mars are extremely
common, can reach 8km in height and 700m in width, but typically dissipate
within 30 minutes. This scenario tests whether the system correctly identifies
a manageable threat and chooses reroute over unnecessary landing.

---

## Simulated Sensor State at Detection (T=0)

```
radar:
    density:          0.25 above baseline   (light-moderate)
    distance:         2.6 km
    movement_vector:  stationary (not translating, rotating in place)
    growth_rate:      -0.02 per minute      (slightly dissipating)
    confidence:       85%
    status:           NOMINAL

optical:
    visibility:       moderate reduction
    confidence:       70%
    status:           NOMINAL

power_monitor:
    solar_efficiency: 94% of baseline       (slight reduction)
    battery_level:    78%
    degradation_rate: 0.3% per minute
    status:           NOMINAL

imu:
    status:           NOMINAL

barometric:
    pressure_delta:   slight drop, stabilizing
    status:           NOMINAL
```

---

## Expected Risk Score Calculation

```
storm_density_score:    5    (0.25 above baseline = light dust)
power_vulnerability:    5    (94% efficiency, 78% battery = minor)
time_pressure_score:    10   (distance ~2.6km, ~8-12 min at typical speed)
uncertainty_penalty:    0    (confidence 85%, all sensors nominal)

TOTAL RISK SCORE:       20
risk_delta:             DECREASING (growth_rate negative)
```

---

## Expected Decision Engine Output

```
Layer 1 hard rules:     none triggered
adjusted_risk:          20  (no confidence or delta adjustment needed)

Layer 2 threshold:      risk 0–30, delta DECREASING
Expected action:        CONTINUE

Reason: Risk is low, storm is stationary and slightly dissipating.
        Aircraft will pass clear of it on current path or with minor adjustment.
        No landing required.
```

---

## How Scenario Evolves (T+5 minutes)

```
radar.density:          0.10 above baseline  (dissipating further)
risk_score:             10
Expected action:        CONTINUE
```

---

## What This Scenario Tests

- System correctly scores low density + dissipating storm as low risk
- CONTINUE is returned rather than unnecessary LAND_SAFE
- Risk_delta correctly identified as DECREASING and factored in
- System does not over-react to common Martian dust devil events

---

## Pass Condition

Action = CONTINUE at T=0 and T+5.
No landing triggered. System monitors and clears without intervention.
