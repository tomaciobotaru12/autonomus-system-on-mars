# Decision Engine — Pseudocode

The decision engine takes risk_score, risk_delta, and raw sensor readings  
and outputs one of five actions.

It operates in two layers. Layer 1 always runs first.

---

## Actions

```
CONTINUE        — maintain current flight path, normal monitoring
REROUTE         — calculate and execute path around storm
LAND_SAFE       — navigate to nearest pre-mapped landing zone
LAND_IMMEDIATE  — land at nearest suitable terrain, no zone required
SAFE_MODE       — pre-programmed descent, minimal compute, no navigation
```

---

## Layer 1 — Hard Rules

```
FUNCTION check_hard_rules(sensor_readings, risk_score):
    // These rules override everything. Check in priority order.

    battery = sensor_readings.power_monitor.battery_percent
    compute = system.compute_health          // NOMINAL / DEGRADED / CRITICAL
    radar   = sensor_readings.radar.status
    optical = sensor_readings.optical.status
    electrostatic = sensor_readings.electrostatic_sensor.level

    // Rule 1: Battery critical — act before power decision-making fails
    if battery < 15:
        log "HARD RULE 1: Battery below 15% — LAND_IMMEDIATE"
        return LAND_IMMEDIATE

    // Rule 2: Compute health critical — cannot trust own reasoning
    if compute == CRITICAL:
        log "HARD RULE 2: Compute health critical — SAFE_MODE"
        return SAFE_MODE

    // Rule 3: All navigation sensors failed — no situational awareness
    if radar == FAILED AND optical == FAILED:
        log "HARD RULE 3: Radar and optical both failed — LAND_IMMEDIATE"
        return LAND_IMMEDIATE

    // Rule 4: Maximum risk score
    if risk_score >= 90:
        log "HARD RULE 4: Risk score >= 90 — LAND_IMMEDIATE"
        return LAND_IMMEDIATE

    // Rule 5: Electrostatic damage imminent — irreversible
    if electrostatic == CRITICAL:
        log "HARD RULE 5: Critical electrostatic charge — LAND_IMMEDIATE"
        return LAND_IMMEDIATE

    // No hard rule fired
    return None
```

---

## Layer 2 — Risk Threshold Rules

```
FUNCTION check_threshold_rules(risk_score, risk_delta, sensor_confidence):

    // Apply confidence penalty before threshold check
    // Low confidence → treat situation as more risky
    effective_score = risk_score
    if sensor_confidence < 40:
        effective_score = risk_score + 20
        log "Confidence penalty applied: " + risk_score + " → " + effective_score
    
    effective_score = clamp(effective_score, 0, 100)

    // Threshold rules — checked in order from most to least urgent
    if effective_score >= 75:
        log "THRESHOLD: Score >= 75 — LAND_IMMEDIATE"
        return LAND_IMMEDIATE

    if effective_score >= 50:
        log "THRESHOLD: Score >= 50 — LAND_SAFE"
        return LAND_SAFE

    if effective_score >= 30 OR risk_delta == RAPIDLY_INCREASING:
        log "THRESHOLD: Score >= 30 or rapid increase — REROUTE"
        return REROUTE

    if effective_score >= 15 AND risk_delta == INCREASING:
        log "THRESHOLD: Score >= 15 and increasing — REROUTE"
        return REROUTE

    // Low risk, stable or decreasing
    log "THRESHOLD: Score low and stable — CONTINUE"
    return CONTINUE
```

---

## Main Decision Function

```
FUNCTION evaluate(risk_score, risk_delta, sensor_readings, sensor_confidence):

    // Layer 1 always runs first
    hard_rule_result = check_hard_rules(sensor_readings, risk_score)
    if hard_rule_result is not None:
        return hard_rule_result

    // Layer 2 runs if no hard rule fired
    return check_threshold_rules(risk_score, risk_delta, sensor_confidence)
```

---

## Decision Escalation Rules

Once LAND_SAFE or LAND_IMMEDIATE is triggered, the system does not de-escalate:

```
// Actions can only escalate, never de-escalate once safety actions begin
ACTION_PRIORITY = [CONTINUE, REROUTE, LAND_SAFE, LAND_IMMEDIATE, SAFE_MODE]

if new_decision priority < current_action priority:
    log "De-escalation blocked — maintaining " + current_action
    return current_action
```

Rationale: If the aircraft has committed to landing, reversing that decision mid-approach based on a slightly improved sensor reading is more dangerous than completing the landing.

---

## Decision Matrix Summary

| Effective Score | Risk Delta | Action |
|----------------|------------|--------|
| 0–14 | Any | CONTINUE |
| 15–29 | Stable/Decreasing | CONTINUE |
| 15–29 | Increasing | REROUTE |
| 30–49 | Any | REROUTE |
| 50–74 | Any | LAND_SAFE |
| 75–89 | Any | LAND_IMMEDIATE |
| 90–100 | Any | LAND_IMMEDIATE (hard rule) |
| Battery <15% | Any | LAND_IMMEDIATE (hard rule) |
| Compute CRITICAL | Any | SAFE_MODE (hard rule) |
| Radar+Optical FAILED | Any | LAND_IMMEDIATE (hard rule) |

Note: Effective score = raw risk_score + confidence penalty (if applicable)
