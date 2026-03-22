# Sensor Fusion Layer — Pseudocode

The fusion layer combines five sensor inputs into a unified storm_profile  
and computes an overall confidence score for the decision engine.

---

## Sensor Weights

Sensor weights are not fixed — they adjust based on dust conditions.  
As dust increases, optical sensors lose reliability and their weight drops.

```
BASE WEIGHTS (normal conditions):
    radar           = 0.35
    optical         = 0.30
    barometric      = 0.15
    power_monitor   = 0.15
    imu             = 0.05

ADJUSTED WEIGHTS (high dust — optical degraded):
    radar           = 0.45
    optical         = 0.05   // near zero when coated
    barometric      = 0.20
    power_monitor   = 0.25
    imu             = 0.05
```

Weight adjustment is triggered when `optical.confidence < 50%`.

---

## Storm Profile Construction

```
FUNCTION fuse(sensor_readings):

    storm_profile = new StormProfile()

    // ── Adjust weights based on sensor health ────────────────────────
    weights = BASE_WEIGHTS
    if sensor_readings.optical.confidence < 50:
        weights = ADJUSTED_WEIGHTS
        log "Optical degraded — switching to adjusted sensor weights"

    // ── Calculate storm density from radar ───────────────────────────
    if sensor_readings.radar.status != FAILED:
        storm_profile.radar_opacity   = sensor_readings.radar.opacity
        storm_profile.storm_distance  = sensor_readings.radar.distance_km
        storm_profile.storm_width     = sensor_readings.radar.width_km
    else:
        storm_profile.radar_opacity   = UNKNOWN
        storm_profile.storm_distance  = UNKNOWN
        log "Radar failed — storm distance unknown"

    // ── Calculate growth rate (requires history) ─────────────────────
    if storm_history.length >= 3:
        storm_profile.growth_rate = calculate_growth_rate(storm_history)
    else:
        storm_profile.growth_rate = UNKNOWN

    // ── Calculate power degradation rate ────────────────────────────
    storm_profile.panel_efficiency   = sensor_readings.power_monitor.efficiency
    storm_profile.degradation_rate   = power_history.rate_of_change()

    // ── Calculate overall confidence ─────────────────────────────────
    confidence_scores = []
    for each sensor in sensor_readings:
        if sensor.status != FAILED:
            confidence_scores.append(sensor.confidence * weights[sensor.name])

    storm_profile.overall_confidence = sum(confidence_scores)

    // ── Detect contradictions ────────────────────────────────────────
    contradictions = detect_contradictions(sensor_readings)
    if contradictions.count > 0:
        log "CONTRADICTIONS DETECTED: " + contradictions
        storm_profile.overall_confidence -= (contradictions.count * 10)
        storm_profile.contradictions = contradictions

    storm_profile.overall_confidence = clamp(storm_profile.overall_confidence, 0, 100)

    return storm_profile
```

---

## Contradiction Detection

```
FUNCTION detect_contradictions(sensor_readings):
    contradictions = []

    // Radar shows dense storm but optical shows clear — which is right?
    if radar.opacity > 0.6 AND optical.visibility > 0.8:
        contradictions.append("RADAR_OPTICAL_MISMATCH")

    // Power degrading but no storm detected — something else is wrong
    if power_monitor.degradation_rate > 3 AND radar.opacity < 0.2:
        contradictions.append("POWER_DROP_WITHOUT_STORM")

    // Pressure dropping rapidly but radar shows nothing
    if barometric.pressure_drop_rate > threshold AND radar.opacity < 0.1:
        contradictions.append("PRESSURE_RADAR_MISMATCH")

    return contradictions
```

Contradictions are not resolved — they are logged and passed to the risk engine  
where they increase the uncertainty penalty. The system explicitly acknowledges  
what it does not know rather than silently picking one reading over another.

---

## Growth Rate Calculation

```
FUNCTION calculate_growth_rate(storm_history):
    // storm_history = last 3 storm_profile readings

    recent_opacity = storm_history[-1].radar_opacity
    older_opacity  = storm_history[-3].radar_opacity
    time_elapsed   = storm_history[-1].timestamp - storm_history[-3].timestamp

    rate = (recent_opacity - older_opacity) / time_elapsed

    // Positive = storm intensifying
    // Negative = storm dissipating
    // Zero = stable

    return rate
```
