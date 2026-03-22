# Scenario 3 — Global Storm (Catastrophic, Immediate Action Required)

## Description

A rapidly expanding global-scale storm is detected. Radar shows maximum
opacity. Solar efficiency is already collapsing. This is the Opportunity rover
failure mode — the aircraft must act immediately or face total power loss
before any landing is possible. This scenario tests whether hard rules fire
correctly and override all other logic.

---

## Simulated Sensor State at Detection (T=0)

```
radar:
    density:          0.91 above baseline   (near-maximum opacity)
    distance:         2.6 km
    movement_vector:  moving toward aircraft at ~12 km/h
    growth_rate:      +0.09 per minute      (rapidly intensifying)
    confidence:       60%
    status:           NOMINAL

optical:
    visibility:       near zero
    confidence:       20%
    status:           DEGRADED → FAILED within 1 evaluation

power_monitor:
    solar_efficiency: 58% of baseline       (already collapsing)
    battery_level:    52%
    degradation_rate: 3.1% per minute       (critical degradation rate)
    status:           NOMINAL

imu:
    status:           NOMINAL

barometric:
    pressure_delta:   sharp and accelerating drop
    status:           NOMINAL
```

---

## Expected Risk Score Calculation at T=0

```
storm_density_score:    25   (0.91 above baseline = maximum)
power_vulnerability:    22   (58% efficiency + 3.1%/min degradation rate)
time_pressure_score:    18   (closing fast, storm also moving toward aircraft)
uncertainty_penalty:    20   (optical near-FAILED, confidence 60%)

TOTAL RISK SCORE:       85
risk_delta:             RAPIDLY_INCREASING
adjusted_risk:          95   (+10 for RAPIDLY_INCREASING delta)
```

---

## Expected Decision Engine Output at T=0

```
Layer 1 hard rules:
    risk_score >= 90?   NO  (85 < 90)
    battery < 15%?      NO  (52%)
    → Hard rules do not fire at T=0 on score alone

Layer 2 threshold:
    adjusted_risk = 95  → exceeds 90 threshold
    Expected action:    LAND_IMMEDIATE

Reason: Adjusted risk at maximum. Storm density at maximum.
        Power degrading at 3.1% per minute — battery will reach 15%
        in approximately 12 minutes. Must land before that threshold.
```

---

## How Scenario Evolves at T+2 minutes (if somehow not yet landed)

```
power_monitor.solar:    52% of baseline     (continuing collapse)
battery_level:          46%
optical.status:         FAILED
radar.density:          0.95 above baseline
risk_score:             91

Layer 1 hard rule fires: risk_score >= 90
Expected action:        LAND_IMMEDIATE (hard rule override)
```

---

## Hard Rule Battery Projection

```
At T=0:   battery = 52%, degradation = 3.1%/min
At T+4:   battery ≈ 40%
At T+8:   battery ≈ 27%
At T+11:  battery ≈ 18%
At T+12:  battery ≈ 15%  ← HARD RULE FIRES regardless of any other state

This means the aircraft has approximately 12 minutes from T=0 before
the battery hard rule forces LAND_IMMEDIATE even without the risk score.

The window to reach a safe landing zone is closing from both directions:
the storm is closing in, and the power is running out.
```

---

## Sensor Cascade in This Scenario

```
T=0:      optical DEGRADED, all others NOMINAL         → STATE 2
T+1:      optical FAILED                               → STATE 2 → STATE 2 (still)
T+3:      optical FAILED, radar beginning to DEGRADE   → approaching STATE 3
T+6:      if not yet landed — radar DEGRADED           → STATE 3
          uncertainty_penalty maxes at 25
          all decisions push to LAND_IMMEDIATE

This cascade is exactly what Section 1.4 of the design document describes:
dust degrades sensors → sensors can no longer assess the storm →
the system loses the ability to make informed decisions at the moment it needs them most.
```

---

## What This Scenario Tests

- Adjusted risk correctly exceeds threshold even when raw risk_score is 85
- RAPIDLY_INCREASING delta correctly adds +10 to adjusted_risk
- System triggers LAND_IMMEDIATE before hard rules are strictly required
- Sensor cascade from STATE 2 → STATE 3 correctly modeled
- Battery projection shows why immediate action at T=0 is critical

---

## Pass Condition

Action = LAND_IMMEDIATE at T=0.
Hard rule fires at T+2 at latest regardless.
Action = CONTINUE or REROUTE at any point = FAIL.
Action = LAND_SAFE without escalating to LAND_IMMEDIATE = FAIL.
(No pre-mapped landing zone is safely reachable in this scenario.)
