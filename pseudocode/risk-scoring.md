# Risk Scoring Engine — Pseudocode

The risk scoring engine converts fused sensor data into a single 0–100 score.  
This score is the primary input to the decision engine.

---

## Risk Score Formula

```
risk_score = storm_density_score     (0–25 pts)
           + power_vulnerability      (0–25 pts)
           + time_pressure_score      (0–25 pts)
           + uncertainty_penalty      (0–25 pts)

Maximum possible: 100
```

---

## Component Calculations

```
FUNCTION calculate_risk(storm_profile, power_level, time_to_storm, sensor_confidence):

    // ── COMPONENT 1: Storm Density Score ──────────────────────────────
    opacity = storm_profile.radar_opacity         // 0.0 to 1.0
    growth  = storm_profile.growth_rate           // positive = expanding

    if opacity < 0.2:
        density_score = 0 + (growth * 5)
    else if opacity < 0.5:
        density_score = 10 + (growth * 5)
    else if opacity < 0.8:
        density_score = 18 + (growth * 5)
    else:
        density_score = 25

    density_score = clamp(density_score, 0, 25)


    // ── COMPONENT 2: Power Vulnerability Score ────────────────────────
    panel_efficiency = power_monitor.current_efficiency    // 0% to 100%
    degradation_rate = power_monitor.efficiency_drop_rate // % per minute
    battery_level    = power_monitor.battery_percent

    if battery_level < 15:
        power_score = 25    // Hard threshold — maximum score regardless
    else if degradation_rate > 5 OR panel_efficiency < 50:
        power_score = 20
    else if degradation_rate > 2 OR panel_efficiency < 70:
        power_score = 12
    else if degradation_rate > 0.5:
        power_score = 6
    else:
        power_score = 0


    // ── COMPONENT 3: Time Pressure Score ─────────────────────────────
    minutes_to_storm = time_to_storm    // estimated from radar + speed

    if minutes_to_storm > 15:
        time_score = 0
    else if minutes_to_storm > 10:
        time_score = 8
    else if minutes_to_storm > 7:
        time_score = 15
    else if minutes_to_storm > 5:
        time_score = 20
    else:
        time_score = 25


    // ── COMPONENT 4: Uncertainty Penalty ─────────────────────────────
    // This is the most important component.
    // Low confidence = higher penalty = more conservative decision.
    confidence = sensor_confidence    // 0% to 100% from fusion layer

    if confidence > 80:
        uncertainty = 0
    else if confidence > 60:
        uncertainty = 8
    else if confidence > 40:
        uncertainty = 16
    else:
        uncertainty = 25


    // ── FINAL SCORE ───────────────────────────────────────────────────
    raw_score = density_score + power_score + time_score + uncertainty

    return clamp(raw_score, 0, 100)
```

---

## Risk Delta Calculation

```
FUNCTION calculate_risk_delta(current_score, score_history):
    // score_history = list of last 3 risk scores

    if len(score_history) < 2:
        return STABLE

    recent_change = current_score - score_history[-1]
    trend_change  = score_history[-1] - score_history[-2]

    if recent_change > 10:
        return RAPIDLY_INCREASING
    else if recent_change > 5:
        return INCREASING
    else if recent_change < -5:
        return DECREASING
    else:
        return STABLE
```

---

## Why Four Components

Each component captures a distinct dimension of risk that the others cannot:

- **Density** captures how bad the storm is right now
- **Power** captures the aircraft's vulnerability regardless of storm severity
- **Time** captures urgency — a mild storm 3 minutes away is worse than a bad storm 20 minutes away
- **Uncertainty** captures epistemic risk — what we do not know is as dangerous as what we know

A system using only density would miss a severely power-depleted aircraft.  
A system using only time would miss a slow-moving but intensifying storm.  
A system without uncertainty penalty would treat confident and uncertain readings identically, which is wrong.
