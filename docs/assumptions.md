# Assumptions

All assumptions made in the design document are listed here explicitly.
Where an assumption significantly affects the solution, an alternative is noted.

---

## Aircraft Design Assumptions

| # | Assumption | Impact if Wrong |
|---|-----------|-----------------|
| A1 | Aircraft is solar powered | If nuclear powered, power loss threat is eliminated entirely. Component 2 of the risk score would be removed and the decision thresholds recalibrated. |
| A2 | Aircraft has onboard radar | Without radar, storm density cannot be assessed through dust. System would rely on barometric and power degradation signals only, and confidence scores would be significantly lower across all conditions. |
| A3 | Aircraft has IMU for inertial navigation | Without IMU, total optical failure = total navigation failure. LAND_IMMEDIATE hard rule must trigger even earlier. |
| A4 | Aircraft has sufficient battery reserve for ~15 minutes of flight after solar degradation begins | If reserve is shorter, all time thresholds in the decision engine compress accordingly. The 15% battery hard rule threshold may need to increase. |
| A5 | Aircraft can execute a controlled landing autonomously | If landing requires external guidance, LAND_SAFE and LAND_IMMEDIATE become infeasible without pre-programmed descent subroutines baked into firmware. |
| A6 | Aircraft has a self-diagnostic system that can report compute health status | Without this, the SAFE_MODE trigger based on compute_health = CRITICAL is unavailable. Hardware watchdog timer becomes the only failsafe. |

---

## Environment Assumptions

| # | Assumption | Impact if Wrong |
|---|-----------|-----------------|
| E1 | Storm is moving toward the aircraft or aircraft is moving toward storm | If storm is moving away, risk_delta would naturally decrease and system would return to CONTINUE without intervention. No design change needed. |
| E2 | Dust storm detected at 12 minutes away is a real storm signal, not sensor noise | The uncertainty penalty handles this — low confidence readings trigger conservative action regardless of whether the threat is real. A false positive costs one landing. |
| E3 | Electrostatic damage to electronics accumulates over minutes, not seconds | If damage is instantaneous upon contact, no decision logic can respond fast enough. In that scenario, hardware shielding of the electronics compartment becomes the only meaningful defense. |
| E4 | The pre-flight baseline atmospheric readings represent normal conditions | If baseline is recorded during a pre-existing dust event, all change detection will be miscalibrated. Baseline recording must be done under verified clear conditions. |

---

## Mission Planning Assumptions

| # | Assumption | Impact if Wrong |
|---|-----------|-----------------|
| M1 | Pre-mapped landing zones exist along and adjacent to the planned flight path | Without this, LAND_SAFE degrades to LAND_IMMEDIATE everywhere. The pre-flight requirement in Section 2.9 of the design document must be treated as mandatory, not optional. |
| M2 | Baseline atmospheric readings are taken before each flight | Without baseline, risk scoring measures absolute values rather than change, making it vulnerable to seasonal dust increases being misread as storm threats and vice versa. |
| M3 | Flight path is planned over terrain the aircraft has sufficient altitude to reroute above | Rerouting over unknown terrain at low altitude is dangerous. Mission planning must validate reroute corridors exist before approving any flight plan. |
| M4 | Pre-mapped landing zones have been surveyed for flatness and approach vectors | A landing zone entry in the database is only useful if the terrain data is accurate. Zones must be validated by orbital imagery or prior rover survey before being added. |

---

## Decision Logic Assumptions

| # | Assumption | Impact if Wrong |
|---|-----------|-----------------|
| D1 | A conservative default (land rather than continue) is always preferable when uncertain | If mission objectives make landing extremely costly, thresholds could be adjusted upward — but the asymmetry of permanently losing the aircraft justifies the conservative default in all standard mission profiles. |
| D2 | 15% battery threshold for the hard LAND_IMMEDIATE rule is sufficient margin for controlled landing | This value requires empirical testing with the specific aircraft design. The rule structure is correct; the threshold value is a tunable parameter. |
| D3 | Risk score thresholds (30, 50, 75, 90) are reasonable starting values | All thresholds are configurable parameters. They should be validated and tuned through simulation testing before deployment. No threshold value in this document should be treated as final. |
| D4 | LAND_IMMEDIATE is irreversible once triggered | This is a deliberate design choice, not a limitation. A system that second-guesses an emergency landing in progress is more dangerous than one that commits. |
| D5 | The 30-second base evaluation interval provides sufficient responsiveness | At 12 minutes to storm contact and a 30-second loop, the system evaluates approximately 24 times before contact. If the aircraft is faster or the storm is faster, this interval should be reduced. |
