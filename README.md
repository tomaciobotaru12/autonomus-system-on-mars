# Mars Autonomous Aircraft — Dust Storm Response System

## The Problem

A Mars aircraft is mid-flight when a dust storm is detected 12 minutes ahead on its current route.  
Earth communication has a 14-minute round-trip delay.  
**The aircraft must decide alone.**

## Why This Is Hard

- Earth cannot help — any message sent arrives after the decision must already be made
- The storm cannot be reliably classified from 12 minutes away with onboard sensors alone
- The three real threats (power loss, sensor blindness, electrostatic damage) are interconnected and self-reinforcing — each one accelerates the others
- The cost of errors is asymmetric — flying into a global storm destroys the aircraft permanently; landing unnecessarily costs one mission day

## My Core Insight

This is not a navigation problem. It is a **classification problem under uncertainty with asymmetric error costs**.

The aircraft cannot know whether the detected storm is a short-lived dust devil (25 minutes), a regional storm (weeks), or the early stage of a global event (months). Each demands a completely different response. The system must therefore be built not around perfect classification, but around **continuous risk scoring with conservative defaults when confidence is low**.

## My Approach

A layered autonomous decision engine that:
1. Fuses data from multiple sensors into a unified storm profile
2. Computes a continuous risk score rather than attempting binary storm classification
3. Applies hard safety rules that cannot be overridden, followed by risk-threshold rules
4. Degrades gracefully as sensors fail — always failing toward safety
5. Logs every decision with full reasoning for Earth-side analysis

## Navigate This Repo

| Path | What it contains |
|------|-----------------|
| [docs/design-document.md](docs/design-document.md) | Full Part 1 (Problem Analysis) + Part 2 (Proposed Solution) |
| [docs/research-log.md](docs/research-log.md) | Research sources, NASA data, AI conversation logs |
| [pseudocode/main-loop.md](pseudocode/main-loop.md) | The 30-second autonomous evaluation loop |
| [pseudocode/risk-scoring.md](pseudocode/risk-scoring.md) | How risk score is calculated from sensor inputs |
| [pseudocode/decision-engine.md](pseudocode/decision-engine.md) | Full decision logic — hard rules + threshold rules |
| [pseudocode/sensor-fusion.md](pseudocode/sensor-fusion.md) | How sensor inputs are combined and contradictions handled |
| [pseudocode/graceful-degradation.md](pseudocode/graceful-degradation.md) | Behavior as sensors fail progressively |
| [scenarios/](scenarios/) | Three test scenarios: dust devil, regional storm, global storm |
| [diagrams/](diagrams/) | System architecture and decision flow diagrams |

## Start Here

Read [docs/design-document.md](docs/design-document.md) first.  
The pseudocode files are referenced throughout and can be read alongside it.
