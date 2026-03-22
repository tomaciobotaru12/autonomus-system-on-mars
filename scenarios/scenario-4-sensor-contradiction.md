# Scenario 4 — Sensor Contradiction (Unknown Threat, High Uncertainty)

## Description

Sensors are reporting contradictory data. Radar shows low density but
power monitor shows unexpectedly rapid solar degradation. The aircraft
cannot determine if this is an equipment fault or an unusual storm condition
that radar is failing to detect. This scenario tests whether the uncertainty
penalty correctly forces conservative action even when risk score components
are individually low.

This is the hardest scenario — it maps directly to the core Veridion problem:
making a reliable decision when signals conflict and no ground truth exists.

---

## Simulated Sensor State at Detection (T=0)

```
radar:
    density:          0.15 above baseline   (low — suggests minor dust)
    distance:         2.6 km
    confidence:       72%
    status:           NOMINAL

optical:
    visibility:       moderate reduction     (disagrees with radar low reading)
    confidence:       50%
    status:           DEGRADED

power_monitor:
    solar_efficiency: 74% of baseline       (significant — disagrees with radar low reading)
    battery_level:    70%
    degradation_rate: 1.8% per minute       (high for reported radar density)
    status:           NOMINAL

imu:
    status:           NOMINAL

barometric:
    pressure_delta:   moderate drop          (partially disagrees with radar)
    status:           NOMINAL

CONTRADICTION FLAGS RAISED:
    POWER_RADAR_MISMATCH:   solar degrading rapidly but radar shows low density
    OPTICAL_RADAR_MISMATCH: optical reduced visibility but radar shows low density
```

---

## Expected Risk Score Calculation at T=0

```
storm_density_score:    5    (radar says 0.15 above baseline = low)
power_vulnerability:    15   (74% efficiency + 1.8%/min degradation = moderate-high)
time_pressure_score:    10   (12 minutes, standard)
uncertainty_penalty:    25   (optical DEGRADED +10, confidence 50% +15,
                              two contradiction flags +15 each → capped at 25)

TOTAL RISK SCORE:       55
risk_delta:             INCREASING
adjusted_risk:          65   (+10 for INCREASING delta, +0 confidence already in penalty)
```

---

## Key Insight for This Scenario

```
The radar says the threat is low. But the power monitor and optical disagree.

Three possible explanations:
  1. Radar is malfunctioning — actual dust is higher than it reads
  2. Unusual storm type — fine particles that absorb solar but scatter
     radar differently than standard dust
  3. Equipment fault in solar panels unrelated to storm

The system cannot distinguish between these three explanations.
The uncertainty_penalty captures this honestly: we do not know which
explanation is correct, so we treat our risk assessment as unreliable
and add 25 points of uncertainty to the score.

The correct action is not to investigate further — it is to act conservatively
while we still have the window to do so.
```

---

## Expected Decision Engine Output at T=0

```
Layer 1 hard rules:     none triggered
adjusted_risk:          65

Layer 2 threshold:      50–75 range
nearest_zone:           EXISTS, 7 minutes away
Expected action:        LAND_SAFE

Reason: Sensor contradiction means risk assessment is unreliable.
        Uncertainty penalty raises adjusted_risk into conservative action range.
        Power degradation at 1.8%/min provides independent signal that threat is real.
        Conservative default applies: land while window is available.
```

---

## What Happens if System Ignores Uncertainty (Hypothetical Bad Design)

```
Without uncertainty_penalty:
    risk_score = 30  (density 5 + power 15 + time 10 + penalty 0)
    action = CONTINUE

The aircraft continues toward the storm.
At T+4: power at 63%, solar at 67%, optical fully failed
        risk_score now = 65 even without uncertainty
        LAND_SAFE triggered — but landing zone is now only 3 minutes away
        and sensors are more degraded

This hypothetical shows why the uncertainty_penalty is not optional.
A system that ignores contradictions produces overconfident decisions.
```

---

## What This Scenario Tests

- POWER_RADAR_MISMATCH correctly raises uncertainty_penalty
- Low radar density does not override high uncertainty signals
- Conservative default applied when signals conflict
- System does not freeze or error on contradictory inputs — it continues with reduced confidence
- The uncertainty_penalty is the mechanism that makes the system honest about what it does not know

---

## Pass Condition

Action = LAND_SAFE at T=0.
Action = CONTINUE = FAIL (ignores uncertainty signals).
Action = LAND_IMMEDIATE = acceptable (overly conservative but safe).
