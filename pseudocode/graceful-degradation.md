# Graceful Degradation — Pseudocode

As sensors fail progressively during a storm, the system adapts.  
At every stage, failure moves the system toward more conservative actions —  
never toward more aggressive ones.

---

## Degradation Stages

```
STAGE 0 — ALL SENSORS NOMINAL
    Active sensors: radar, optical, barometric, power_monitor, imu
    Confidence: full
    Behaviour: normal operation, full risk score calculation

STAGE 1 — OPTICAL SENSORS DEGRADED
    Trigger: optical.confidence < 50% OR optical.status == DEGRADED
    Active sensors: radar, barometric, power_monitor, imu
    Optical weight → near zero in fusion layer
    Uncertainty penalty automatically increases
    Effect: effective risk score rises, system becomes more conservative
    Log: "DEGRADATION STAGE 1: Optical sensors degraded"

STAGE 2 — OPTICAL SENSORS FAILED
    Trigger: optical.status == FAILED
    Active sensors: radar, barometric, power_monitor, imu
    Navigation relies on radar + IMU only
    Uncertainty penalty: +15 points added to raw uncertainty score
    Log: "DEGRADATION STAGE 2: Optical sensors failed"

STAGE 3 — OPTICAL + RADAR FAILED
    Trigger: radar.status == FAILED AND optical.status == FAILED
    Active sensors: barometric, power_monitor, imu
    Cannot assess storm externally — only internal aircraft state available
    Hard Rule 3 fires: LAND_IMMEDIATE triggered
    Navigation for landing: IMU-only descent
    Log: "DEGRADATION STAGE 3: External sensors failed — LAND_IMMEDIATE"

STAGE 4 — COMPUTE DEGRADED
    Trigger: system.compute_health == DEGRADED
    Behaviour: reduce evaluation cycle complexity
               skip growth rate calculation (requires history)
               use simplified risk formula
               increase telemetry frequency
    Log: "DEGRADATION STAGE 4: Compute degraded — simplified mode"

STAGE 5 — COMPUTE CRITICAL / ALL SENSORS FAILED
    Trigger: system.compute_health == CRITICAL
             OR all external sensors FAILED
             OR battery < 10%
    Behaviour: SAFE_MODE activated
               pre-programmed controlled descent
               no navigation, no risk calculation
               maximum power conservation
               continuous black box transmission until power loss
    Log: "DEGRADATION STAGE 5: SAFE_MODE — pre-programmed descent"
```

---

## Degradation Response Function

```
FUNCTION handle_degradation(sensor_readings, current_stage):

    new_stage = assess_degradation_stage(sensor_readings)

    // Stages only increase — never decrease during a storm event
    if new_stage > current_stage:
        log "STAGE CHANGE: " + current_stage + " → " + new_stage
        apply_stage_response(new_stage)
        return new_stage

    return current_stage


FUNCTION apply_stage_response(stage):

    if stage == 1:
        sensor_fusion.adjust_weights(ADJUSTED_WEIGHTS)
        risk_engine.increase_uncertainty_baseline(10)

    if stage == 2:
        sensor_fusion.adjust_weights(RADAR_ONLY_WEIGHTS)
        risk_engine.increase_uncertainty_baseline(15)

    if stage == 3:
        // Hard rule will fire in decision engine
        // Prepare IMU-only landing navigation
        navigation.switch_to_imu_only()

    if stage == 4:
        evaluation.switch_to_simplified_mode()
        telemetry.increase_frequency()

    if stage == 5:
        execute(SAFE_MODE)
```

---

## Design Principle

The degradation model ensures:

1. **No sensor failure causes increased risk tolerance** — every failure increases conservatism
2. **No silent failures** — every degradation event is logged and transmitted
3. **The system always has a next action** — even with all sensors failed, SAFE_MODE provides controlled descent
4. **Human engineers can reconstruct exactly what happened** — full degradation log preserved in black box

The system is designed so that the worst case is a controlled landing with no navigation data —  
not a crash or an uncontrolled descent.
