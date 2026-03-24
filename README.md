# Mars Autonomous Aircraft — Dust Storm Response System

## The Problem

A Mars aircraft is mid-flight when a dust storm is detected 12 minutes ahead on its current route.  
Earth communication has a 14-minute round-trip delay.  
**The aircraft must decide alone.**

## Why This Is Hard

- Earth cannot help, any message sent arrives after the decision must already be made
- The storm cannot be reliably classified from 12 minutes away with onboard sensors alone
- The three real threats (power loss, sensor blindness, electrostatic damage) are interconnected and self-reinforcing, each one accelerates the others
- The cost of errors is asymmetric, flying into a global storm destroys the aircraft permanently; landing unnecessarily costs one mission day

## My Core Insight

This is not a navigation problem. It is a **decision problem under uncertainty with asymmetric error costs**.

The aircraft cannot know whether the detected storm is a short-lived dust devil (25 minutes), a regional storm (weeks), or the early stage of a global event (months). Each demands a completely different response. The system must therefore be built not around perfect classification, but around **continuous risk scoring with conservative defaults when confidence is low**.

## My Approach

A layered autonomous decision engine that:
1. Fuses data from multiple sensors into a unified storm profile
2. Computes a continuous risk score rather than attempting storm type classification
3. Applies hard safety rules that cannot be overridden, followed by risk-threshold rules
4. Degrades gracefully as sensors fail and always failing toward safety, never toward confidence
5. Logs every decision with full reasoning for Earth-side post-flight analysis

---

## Navigate This Repo

### Documentation
| File | What it contains |
|------|-----------------|
| [docs/design-document.md](docs/design-document.md) | Full Part 1 (Problem Analysis) + Part 2 (Proposed Solution) — start here |
| [docs/research-log.md](docs/research-log.md) | NASA sources, real mission references (Ingenuity, Opportunity), AI tool usage |
| [docs/assumptions.md](docs/assumptions.md) | Every assumption made explicitly listed with impact if wrong |

### Decision Logic (Pseudocode)
| File | What it contains |
|------|-----------------|
| [pseudocode/main-loop.md](pseudocode/main-loop.md) | The top-level 30-second evaluation loop that orchestrates all components |
| [pseudocode/sensor-fusion.md](pseudocode/sensor-fusion.md) | How five sensor inputs are weighted, combined, and contradiction-checked |
| [pseudocode/risk-scoring.md](pseudocode/risk-scoring.md) | The four-component risk formula that produces a 0–100 risk score |
| [pseudocode/decision-engine.md](pseudocode/decision-engine.md) | Hard rules (non-negotiable) + risk threshold rules that produce the final action |
| [pseudocode/graceful-degradation.md](pseudocode/graceful-degradation.md) | The five system states as sensors fail — defines exactly what happens at each stage |

### Test Scenarios
| File | What it tests |
|------|--------------|
| [scenarios/scenario-dust-devil.md](scenarios/scenario-dust-devil.md) | Short-lived localized vortex — system should reroute, not land |
| [scenarios/scenario-regional-storm.md](scenarios/scenario-regional-storm.md) | Multi-day regional storm with growing risk delta — system should land safe |
| [scenarios/scenario-global-storm.md](scenarios/scenario-global-storm.md) | Planet-encircling event with rapid power degradation — system should land immediately |

Each scenario walks through the full decision loop step by step, showing exact sensor inputs, risk score calculation, and expected output at each evaluation interval. They are designed to verify that the decision engine behaves correctly at each risk threshold — and that edge cases (sensor contradictions, no landing zone available, low confidence) are handled without producing dangerously overconfident decisions.

### Diagrams
| File | What it shows |
|------|--------------|
| [diagrams/system-architecture.png](diagrams/system-architecture.png) | The five-layer data flow: Sensor → Fusion → Risk → Decision → Action |
| [diagrams/decision-flow.png](diagrams/decision-flow.png) | Full decision tree from storm detection through every possible action |
| [diagrams/threat-cascade.png](diagrams/threat-cascade.png) | How the three threats (power loss, sensor blindness, electrostatic damage) amplify each other |

The threat cascade diagram is particularly important — it shows why the system must act before any single threshold is crossed, not after. Each threat accelerates the others, meaning the window for a safe response shrinks faster than any individual metric would suggest.

---

## Reading Order

1. **[docs/design-document.md](docs/design-document.md)** — Read Part 1 fully before Part 2. The solution only makes sense once the problem is understood precisely.
2. **[diagrams/system-architecture.png](diagrams/system-architecture.png)** — Get a visual map of how the components connect.
3. **[pseudocode/main-loop.md](pseudocode/main-loop.md)** — See how the loop orchestrates everything.
4. **[pseudocode/decision-engine.md](pseudocode/decision-engine.md)** — The core logic. Hard rules first, threshold rules second.
5. **[scenarios/](scenarios/)** — Run through the three scenarios to see the system in action.

---


