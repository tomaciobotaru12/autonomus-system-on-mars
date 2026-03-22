# Design Document — Mars Autonomous Aircraft Dust Storm Response

**Author:** Ciobotaru Toma  
**Date:** March 2026  
**Context:** Engineering design assignment — Veridion

---

## Table of Contents

1. [Part 1 — Problem Analysis](#part-1--problem-analysis)
   - [The Scenario](#the-scenario)
   - [Understanding the Environment](#1-understanding-the-environment)
   - [Understanding the Threats](#2-understanding-the-threats)
   - [The Threat Cascade](#3-the-threat-cascade)
   - [Understanding the Decision Window](#4-understanding-the-decision-window)
   - [What Makes This Problem Hard](#5-what-makes-this-problem-hard)
   - [Key Assumptions](#6-key-assumptions)
   - [Open Questions](#7-open-questions)

2. [Part 2 — Proposed Solution](#part-2--proposed-solution)
   - [Design Philosophy](#design-philosophy)
   - [System Architecture](#system-architecture)
   - [Sensor Layer](#1-sensor-layer)
   - [Sensor Fusion Layer](#2-sensor-fusion-layer)
   - [Risk Scoring Engine](#3-risk-scoring-engine)
   - [Decision Engine](#4-decision-engine)
   - [Action Executor](#5-action-executor)
   - [Telemetry and Black Box](#6-telemetry-and-black-box)
   - [Graceful Degradation](#7-graceful-degradation)
   - [Pre-Mapped Landing Zones](#8-pre-mapped-landing-zones)
   - [Why Rule-Based Over ML](#9-why-rule-based-over-ml)
   - [Implementation Order](#10-implementation-order)
   - [The Broader Principle](#the-broader-principle)

---

# Part 1 — Problem Analysis

## The Scenario

An autonomous aircraft operating on Mars detects a dust storm approximately 12 minutes ahead on its current route. Communication with Earth has a 14-minute round-trip delay. The aircraft must decide what to do without any possibility of consulting Earth before the decision must be made.

Before proposing any solution, I spent time understanding what this scenario actually involves. Below is everything I found that needs to be understood.

---

## 1. Understanding the Environment

### Mars Atmosphere

Mars has an atmosphere approximately 1% the density of Earth's. This has a counterintuitive implication for the scenario: a dust storm moving at 13 km/h on Mars exerts almost no physical force on an aircraft. The wind itself is not the danger.

This is fundamentally different from how we intuitively think about storms. The threat model must be completely reframed.

### Mars Dust Properties

Mars dust particles average 3–10 micrometers in diameter — five to ten times finer than talcum powder. Due to Mars' lower gravity and thin atmosphere, these particles stay suspended in the air for weeks to months once lofted. The particles are highly electrostatically charged, causing them to adhere to surfaces with unusual persistence — similar to how styrofoam sticks to hands due to static electricity, but on critical electronic and optical surfaces.

### Storm Types and Scale

Mars dust storms exist at three scales, and this distinction is critical:

- **Dust devils:** Localized convective vortices, up to 8 km tall and 700 m wide, lasting 25–30 minutes. Common, frequently occurring, generally survivable.
- **Local/Regional storms:** Can cover areas the size of a continent, lasting days to weeks.
- **Global storms:** Occur roughly every 3–4 Mars years, engulf the entire planet, can reduce solar illumination to 1% for up to three months. The 2018 global dust event killed the Opportunity rover.

**Critically: storms at any scale can grow.** A local storm that looks manageable at detection time may expand to regional scale within hours. Scientists with orbital data and years of experience cannot reliably predict which storms will grow. An onboard system with 12 minutes of sensor data certainly cannot.

### Real Precedent: Ingenuity Helicopter

NASA's Ingenuity helicopter operated autonomously on Mars from 2021 to 2024. It survived dust storms by managing power autonomously, performed three emergency landings, and was upgraded mid-mission to autonomously select landing sites in treacherous terrain. Its failure mode during dust periods was instructive: increased atmospheric dust reduced solar charging, causing the flight computer's clock to reset and heaters to switch off — a cascade that began with dust and ended with the computer losing track of time.

---

## 2. Understanding the Threats

Research revealed that the threats from a Mars dust storm are **not wind force** but three interconnected vectors:

### Threat 1: Solar Power Loss

Dust particles coat solar panels, reducing energy generation. During the 2018 global storm, atmospheric opacity (measured as "tau") reached 4.7, meaning approximately 99% of sunlight was blocked. This is what killed Opportunity. For a solar-powered aircraft, power loss is the most critical threat because it is the foundation everything else depends on — no power means no sensors, no compute, no flight control.

Solar degradation happens gradually, but the rate of degradation can accelerate rapidly as the storm intensifies. The aircraft cannot wait to observe significant degradation before acting — by then it may lack the power to execute a controlled response.

### Threat 2: Sensor and Navigation Blindness

Dust particles coat optical sensors and cameras progressively. Once optical sensors degrade, the aircraft loses its primary navigation capability. Radar and IMU (inertial measurement unit) survive longer but provide increasingly incomplete situational awareness. At some threshold of sensor degradation, the aircraft can no longer assess its own situation — it becomes blind to the very data it needs to make decisions.

This creates a dangerous feedback loop: the storm degrades the sensors needed to measure the storm's severity.

### Threat 3: Electrostatic Damage

Mars dust carries strong electrostatic charge. This charge can cause short circuits in exposed electronics and permanently damage sensors and avionics components. Unlike power loss, electrostatic damage is **irreversible**. A decision made too late — after significant electrostatic exposure — cannot be undone even if the aircraft lands safely.

---

## 3. The Threat Cascade

The three threats are not independent. They form a self-reinforcing cascade:

```
Dust enters atmosphere
        ↓
Solar panels coated → Power generation drops
        ↓
Less power → Reduced compute and sensor capacity
        ↓
Degraded sensors → Cannot accurately assess storm severity
        ↓
Poor assessment → Decision delayed or incorrect
        ↓
More dust exposure → Electrostatic damage to electronics
        ↓
Electronics degraded → Cannot execute response even if decided
```

**The critical implication:** The decision to act must happen early — before any single threshold is crossed — because each threshold crossed makes the next threshold harder to avoid. Waiting for clear evidence of danger is itself dangerous.

---

## 4. Understanding the Decision Window

The timing creates a specific constraint:

- Storm detected: T = 0 minutes
- Storm arrives at current route: T = 12 minutes
- Earth receives notification: T + 7 minutes (one-way delay)
- Earth can respond: T + 14 minutes (round-trip)
- Storm contact on current route: T + 12 minutes

Earth's earliest possible response arrives **2 minutes after the storm would have been reached** if no action is taken. Earth cannot help with this decision. The aircraft must act, and it must act within the 12-minute window — ideally well within it to preserve options.

---

## 5. What Makes This Problem Hard

The core difficulty is not technical — it is epistemological.

**The aircraft faces a classification problem before it faces a navigation problem.**

The detected storm could be:
- A dust devil lasting 25 minutes that will dissipate harmlessly
- A local storm that can be navigated around
- A regional storm requiring immediate landing
- The early stage of a global event requiring emergency response

Each scenario demands a completely different action. Yet the sensor data available in 12 minutes may be insufficient to distinguish between them with any reliability. Scientists with orbital data and historical records still cannot reliably predict which storms grow.

**The system must therefore decide before the classification is certain, under conditions where the cost of errors is asymmetric:**
- False positive (lands when unnecessary): mission loses one flight day
- False negative (flies into serious storm): aircraft potentially destroyed permanently

This asymmetry means the system should be biased toward caution. But how cautious? Excessive caution means the aircraft never completes missions. Insufficient caution means it gets destroyed.

**This is not a problem with a correct answer. It is a problem that requires principled reasoning about uncertainty.**

A secondary difficulty: the sensors used to assess the storm are degraded by the very phenomenon being assessed. The measurement system is not independent of the thing being measured.

---

## 6. Key Assumptions

I am making the following assumptions explicitly. Changing any of these would require revisiting parts of the solution:

1. The aircraft is **solar powered** — power loss from dust is therefore a critical threat
2. The aircraft has **pre-loaded maps** of its flight area including candidate landing zones surveyed before flight
3. The aircraft has **onboard compute** capable of running the decision logic described below
4. The aircraft has a **suite of sensors** including radar, optical cameras, atmospheric pressure sensor, IMU, and solar panel efficiency monitor
5. The aircraft operates **fully autonomously** — no human-in-the-loop is possible within the relevant time window
6. **All decisions and sensor states are logged** to a local black box and transmitted to Earth continuously
7. The aircraft has **four possible actions:** continue on route, reroute around the storm, land at a pre-mapped safe zone, or land immediately at nearest suitable terrain

---

## 7. Open Questions

These are questions I would ask before implementation begins:

- What is the aircraft's battery reserve capacity — how long can it fly on stored energy if solar input drops to zero?
- Are the aircraft's electronics shielded against electrostatic discharge? If yes, how much shielding?
- What sensors are available and what are their failure modes in dusty conditions?
- How accurate and recent are the pre-loaded terrain maps?
- What is the aircraft's speed relative to storm movement speed?
- Is there a minimum altitude below which flight becomes impossible or unsafe?
- What is the onboard compute's power draw and at what battery level must it be shut down?
- Has the specific sensor suite been tested in Mars dust conditions?

---

# Part 2 — Proposed Solution

## Design Philosophy

The problem analysis reveals that the core challenge is **making reliable decisions from unreliable signals, under time pressure, with asymmetric error costs, and no external authority to consult.**

Two principles follow from this:

**Principle 1: Measure uncertainty explicitly and factor it into decisions.**  
Rather than attempting to classify the storm type (which requires certainty we do not have), compute a continuous risk score that incorporates sensor confidence as a first-class input. Low confidence should automatically increase conservatism.

**Principle 2: When in doubt, default to the more conservative action.**  
The asymmetric cost of errors — lost aircraft vs lost flight day — justifies a conservative bias. The system should err toward landing, not toward continuing.

These two principles drive every architectural decision below.

---

## System Architecture

The system is composed of six layers, each with a single responsibility:

```
┌──────────────────────────────────────────────────┐
│                 SENSOR LAYER                     │
│  Radar | Optical | Power Monitor | IMU | Baro    │
│  Each outputs: value + confidence% + status      │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│             SENSOR FUSION LAYER                  │
│  Weighted combination of sensor inputs           │
│  Contradiction detection and flagging            │
│  Output: storm_profile + overall_confidence      │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│            RISK SCORING ENGINE                   │
│  Four-component weighted formula                 │
│  Output: risk_score (0-100) + risk_delta         │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│              DECISION ENGINE                     │
│  Layer 1: Hard rules (cannot be overridden)      │
│  Layer 2: Risk threshold rules                   │
│  Output: action + full reasoning log entry       │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│              ACTION EXECUTOR                     │
│  CONTINUE | REROUTE | LAND_SAFE | LAND_IMMEDIATE │
│  | SAFE_MODE                                     │
│  Each action is a pre-defined subroutine         │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────────┐
│          TELEMETRY + BLACK BOX                   │
│  Local storage of all decisions and sensor state │
│  Continuous transmission to Earth                │
│  Survives aircraft loss                          │
└──────────────────────────────────────────────────┘
```

See [pseudocode/main-loop.md](../pseudocode/main-loop.md) for the top-level evaluation loop.

---

## 1. Sensor Layer

Five sensors feed the system. Each sensor is treated as an independent source with its own reliability rating:

| Sensor | What it measures | Dust resistance | Failure mode |
|--------|-----------------|-----------------|--------------|
| Radar | Storm distance, density, movement | High — survives dust | Gradual range reduction |
| Optical camera | Visibility, terrain, storm visual | Low — coats quickly | Progressive blindness |
| Atmospheric pressure | Storm approach, pressure changes | High | Generally reliable |
| IMU | Aircraft orientation, position | Very high — internal | Drift over time |
| Power monitor | Solar panel efficiency % | Very high — internal | None expected |

Each sensor outputs three values every cycle:
- `value` — the measured reading
- `confidence` — self-assessed reliability (0–100%), reduced automatically when readings are inconsistent with recent history or known failure modes
- `status` — NOMINAL / DEGRADED / FAILED

See [pseudocode/sensor-fusion.md](../pseudocode/sensor-fusion.md) for full detail.

---

## 2. Sensor Fusion Layer

The fusion layer combines sensor inputs into a single `storm_profile` object. Its key responsibilities:

**Weighted combination:** Sensors with higher dust resistance are weighted more heavily as conditions deteriorate. Early in detection, optical sensors contribute heavily. As dust increases, radar and pressure sensor weight increases automatically.

**Contradiction detection:** If radar indicates a large dense storm but optical sensors show clear conditions, the system flags a contradiction. Contradictions increase the `uncertainty_penalty` in the risk score rather than being silently resolved. The system explicitly acknowledges what it does not know.

**Overall confidence output:** A single confidence score (0–100%) summarising how much the system trusts its current storm assessment. This feeds directly into the risk scoring engine.

---

## 3. Risk Scoring Engine

Rather than classifying the storm into a discrete category, the system computes a continuous risk score from four components:

```
risk_score = storm_density_score
           + power_vulnerability_score
           + time_pressure_score
           + uncertainty_penalty

Maximum: 100 points
```

**Component 1 — Storm Density Score (0–25 points)**
Derived from radar opacity readings and atmospheric pressure drop rate.
- 0–5 pts: light dust, minimal opacity
- 5–15 pts: moderate density, visible opacity
- 15–25 pts: high density, significant opacity

**Component 2 — Power Vulnerability Score (0–25 points)**
Derived from rate of solar panel efficiency degradation.
- 0 pts: no degradation detected
- 10 pts: 10–20% efficiency drop
- 25 pts: >30% efficiency drop or critical battery level

**Component 3 — Time Pressure Score (0–25 points)**
Derived from estimated time until storm contact at current trajectory.
- 0 pts: >15 minutes
- 10 pts: 10–15 minutes
- 20 pts: 5–10 minutes
- 25 pts: <5 minutes

**Component 4 — Uncertainty Penalty (0–25 points)**
Derived from overall sensor confidence score.
- 0 pts: all sensors agree, confidence >80%
- 10 pts: minor contradictions, confidence 60–80%
- 20 pts: significant contradictions, confidence 40–60%
- 25 pts: sensors unreliable, confidence <40%

The uncertainty penalty is the most important design choice. **Low confidence automatically increases the risk score**, which automatically increases conservatism. The system does not need special handling for "I don't know" — uncertainty is priced into the score directly.

Additionally, the engine computes `risk_delta` — whether the risk score is increasing, stable, or decreasing over the last three evaluation cycles. A rapidly increasing risk_delta triggers earlier threshold crossings.

See [pseudocode/risk-scoring.md](../pseudocode/risk-scoring.md) for full detail.

---

## 4. Decision Engine

The decision engine operates in two layers. Layer 1 always runs first and can override everything.

### Layer 1 — Hard Rules

These rules fire regardless of risk score. They represent conditions where the situation is already beyond the point where nuanced risk assessment is useful:

```
IF battery_level < 15%
    → LAND_IMMEDIATE
    (No power means no decision-making — act before this point)

IF compute_health = CRITICAL
    → SAFE_MODE
    (Cannot trust own reasoning — execute pre-programmed safe descent)

IF radar FAILED AND optical FAILED
    → LAND_IMMEDIATE
    (No sensor input means no situational awareness — land now)

IF risk_score > 90
    → LAND_IMMEDIATE
    (Maximum risk — no threshold rule needed)

IF electrostatic_charge_detected = CRITICAL
    → LAND_IMMEDIATE
    (Irreversible damage imminent)
```

### Layer 2 — Risk Threshold Rules

If no hard rule fires, the threshold rules apply:

```
IF risk_score 0–30 AND risk_delta STABLE OR DECREASING
    → CONTINUE
    → Re-evaluate in 30 seconds

IF risk_score 0–30 AND risk_delta INCREASING
    → REROUTE
    → Re-evaluate in 20 seconds

IF risk_score 30–50 OR risk_delta RAPIDLY_INCREASING
    → REROUTE
    → Re-evaluate in 15 seconds

IF risk_score 50–75
    → LAND_SAFE (navigate to nearest pre-mapped landing zone)
    → Re-evaluate every 10 seconds during approach

IF risk_score 75–90
    → LAND_IMMEDIATE (land at nearest suitable terrain now)
    → No further re-evaluation — commit to landing

IF risk_score ANY AND sensor_confidence < 40%
    → Apply +20 point penalty to risk_score before threshold check
    → Low confidence treated as higher risk automatically
```

See [pseudocode/decision-engine.md](../pseudocode/decision-engine.md) for full detail.

---

## 5. Action Executor

Each action is a pre-defined subroutine that runs deterministically once triggered:

**CONTINUE:** Maintain current flight path. Increase monitoring frequency based on risk level.

**REROUTE:** Calculate alternative path around detected storm using pre-loaded terrain maps. Select path that maximises distance from storm while remaining within mapped terrain. Log original route and reroute trigger conditions.

**LAND_SAFE:** Query pre-loaded landing zone database. Select nearest zone above flatness threshold. Execute pre-defined landing approach for that zone. Transmit landing zone selection and ETA to Earth.

**LAND_IMMEDIATE:** Identify nearest flat terrain from radar + IMU data. Execute maximum-priority landing sequence. No route optimisation — prioritise speed of landing over landing zone quality.

**SAFE_MODE:** Pre-programmed controlled descent with no sensor input required. Reduce altitude at maximum safe rate. Minimal power draw. Activate only when compute health is critical.

---

## 6. Telemetry and Black Box

Although Earth cannot intervene in time, telemetry design is critical for two reasons:
1. Earth can monitor the situation and prepare responses for after the event
2. All data is preserved for future mission learning regardless of outcome

Every 30 seconds, the following is transmitted and stored locally:
- Full sensor readings and confidence values
- Current storm_profile
- Current risk_score and risk_delta
- Current decision and reasoning
- Aircraft position, altitude, battery level

Local black box storage is independent of the communication system. Even if the aircraft is lost, the black box transmits its final stored state continuously until power fails.

---

## 7. Graceful Degradation

As sensors fail progressively during a storm, the system adapts rather than failing:

**All sensors nominal:** Full risk score calculation. Normal operation.

**Optical sensors failed (dust coating):**
- Remove optical from fusion calculation
- Uncertainty penalty increases automatically (+15 points)
- Radar, pressure, and power monitor carry full weight
- Decision thresholds effectively lower — system becomes more conservative

**Optical + radar failed:**
- Only power monitor, barometric, and IMU operational
- Uncertainty penalty maxes out (25 points)
- Hard rule likely triggers: LAND_IMMEDIATE
- IMU used for controlled descent navigation

**All external sensors failed:**
- SAFE_MODE activated
- Pre-programmed descent, no navigation
- Maximum power conservation
- Continuous black box transmission until power loss

At every stage, the system fails **toward safety** not toward mission continuation.

See [pseudocode/graceful-degradation.md](../pseudocode/graceful-degradation.md) for full detail.

---

## 8. Pre-Mapped Landing Zones

LAND_SAFE assumes pre-loaded landing zone data. This is a hard design requirement, not an assumption:

Before any flight, the mission planning system must load a database of pre-surveyed landing zones covering the planned route and adjacent terrain within rerouting range. Each entry contains:

- GPS coordinates
- Terrain flatness rating (0–10)
- Maximum approach vector (direction of safest approach)
- Distance from planned flight path
- Last survey timestamp

The decision engine queries this database when LAND_SAFE is triggered, selecting the nearest zone above a minimum flatness threshold. If no suitable zone exists within range, LAND_SAFE automatically escalates to LAND_IMMEDIATE.

This requirement should drive mission planning: **no flight should be approved without adequate landing zone coverage along its route.**

---

## 9. Why Rule-Based Over ML

A machine learning approach might seem appealing for storm classification. It is not appropriate here for three reasons:

**No training data:** Mars aircraft storm encounter data is essentially zero. Ingenuity never flew through a dust storm. ML models trained on insufficient data in safety-critical contexts are dangerous — they produce confident wrong answers.

**Predictability requirements:** Safety-critical autonomous systems require that every failure mode be known and tested in advance. Rule-based systems have enumerable, testable failure modes. ML failure modes are not enumerable.

**No retraining capability:** An ML model that encounters novel conditions cannot be retrained in the field with a 14-minute communication delay. Rule-based logic handles novel inputs predictably — uncertain inputs trigger conservative defaults.

ML can be introduced in future missions once sufficient flight data exists to train on. For this mission, deterministic rule-based logic is the correct choice.

---

## 10. Implementation Order

For an engineer picking this up:

1. **Sensor interface definitions** — data structures for each sensor output (value, confidence, status)
2. **Sensor fusion layer** — weighted combination, contradiction detection, confidence output
3. **Risk scoring engine** — four components, unit tested independently, then integrated
4. **Landing zone database** — data structure, nearest-zone query, flatness filter
5. **Decision engine** — hard rules first (exhaustively tested), then threshold rules
6. **Action executor** — each action as independent subroutine, each testable in isolation
7. **Telemetry and black box** — local storage first, transmission layer second
8. **Main evaluation loop** — ties all layers together, 30-second cycle
9. **Failure injection testing** — kill sensors one by one, verify graceful degradation
10. **Scenario testing** — run three test scenarios, verify decisions match expected outputs

Test scenarios are provided in [scenarios/](../scenarios/).

---

## The Broader Principle

This solution is fundamentally a system for making reliable decisions from unreliable, incomplete, and potentially contradictory signals — with no ground truth available at decision time and no external authority to consult.

The risk scoring approach, confidence weighting, conservative defaults, and graceful degradation design all follow from one recognition: **uncertainty is not an exception to be handled — it is the normal operating condition.**

The same principles apply wherever decisions must be made without a definitive answer key: measure uncertainty explicitly, weight it into decisions, default conservatively when confidence is low, log everything for future learning, and design failure modes that preserve the most important thing even when everything else goes wrong.

On Mars, the most important thing is the aircraft.  
In other domains, it may be data integrity, user trust, or financial safety.  
The architecture is the same.
