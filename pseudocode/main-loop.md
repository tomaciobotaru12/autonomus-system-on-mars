# Main Evaluation Loop — Pseudocode

This is the top-level loop that runs continuously during flight.  
Every cycle, all six layers execute in sequence.

---

## Evaluation Cycle

```
CONSTANTS:
    NORMAL_INTERVAL     = 30 seconds
    ELEVATED_INTERVAL   = 15 seconds
    HIGH_INTERVAL       = 10 seconds
    CRITICAL_INTERVAL   = 5 seconds

ON AIRCRAFT STARTUP:
    load landing_zone_database from pre-flight mission plan
    initialise all sensor interfaces
    set current_action = CONTINUE
    set evaluation_interval = NORMAL_INTERVAL
    start telemetry transmission

ON STORM DETECTED (trigger from radar or atmospheric sensor):
    log "STORM DETECTED — beginning evaluation loop"
    set evaluation_interval = ELEVATED_INTERVAL
    begin evaluation cycle immediately


EVALUATION CYCLE (runs every evaluation_interval seconds):

    STEP 1 — COLLECT SENSOR DATA
        for each sensor in [radar, optical, power_monitor, imu, barometric]:
            reading = sensor.read()
            // Each reading contains: value, confidence, status
            if reading.status == FAILED:
                log "SENSOR FAILED: " + sensor.name
            sensor_readings.append(reading)

    STEP 2 — SENSOR FUSION
        storm_profile = fuse(sensor_readings)
        // storm_profile contains: density, size, growth_rate, distance
        // overall_confidence = weighted average of sensor confidences
        log storm_profile

    STEP 3 — RISK SCORING
        risk_score = calculate_risk(storm_profile, power_level, time_to_storm)
        risk_delta  = risk_score - previous_risk_score
        log risk_score, risk_delta

    STEP 4 — DECISION ENGINE
        decision = evaluate(risk_score, risk_delta, sensor_readings)
        // Decision engine checks hard rules first, then threshold rules
        log decision + full reasoning

    STEP 5 — ACTION EXECUTOR
        if decision != current_action:
            log "ACTION CHANGE: " + current_action + " → " + decision
            execute(decision)
            current_action = decision

    STEP 6 — TELEMETRY
        transmit_to_earth(sensor_readings, storm_profile, risk_score, decision)
        write_to_black_box(full_state_snapshot)

    STEP 7 — ADJUST INTERVAL
        if risk_score > 75:
            evaluation_interval = CRITICAL_INTERVAL
        else if risk_score > 50:
            evaluation_interval = HIGH_INTERVAL
        else if risk_score > 30:
            evaluation_interval = ELEVATED_INTERVAL
        else:
            evaluation_interval = NORMAL_INTERVAL

    STEP 8 — WAIT
        wait(evaluation_interval)
        goto STEP 1
```

---

## Notes

- The loop runs faster as risk increases — more frequent evaluation when stakes are higher
- Steps 1–6 must complete well within the minimum evaluation interval (5 seconds)
- If compute is degraded and cannot complete a cycle in time, SAFE_MODE is triggered automatically
- The loop never terminates during flight — it runs until landing is complete or power fails
