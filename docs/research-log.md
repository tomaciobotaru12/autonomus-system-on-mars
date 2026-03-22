# Research Log

This document records the research process behind the design document, including sources consulted, key findings, and the AI-assisted exploration process.

---

## Sources Consulted

### NASA and Scientific Sources

**Ingenuity Helicopter — Power and Dust**
- NASA JPL: "After Three Years on Mars, NASA's Ingenuity Helicopter Mission Ends"
  - Key finding: Ingenuity survived dust storms by managing power autonomously; batteries failed due to seasonal dust increase reducing solar charging
  - Source: https://www.nasa.gov/news-release/after-three-years-on-mars-nasas-ingenuity-helicopter-mission-ends/

- Space.com: "Mars helicopter Ingenuity recovering from communications blackout spawned by dust"
  - Key finding: Dust caused FPGA power loss → clock reset → heaters off → cascade failure. Confirmed the threat cascade model.
  - Source: https://www.space.com/mars-ingenuity-helicopter-blackout-dust

- IEEE Spectrum: "How NASA Designed a Helicopter That Could Fly Autonomously on Mars"
  - Key finding: Real-time control impossible due to communication delay. Ingenuity already operates with full autonomous flight logic. Rule-based autonomy is proven on Mars.
  - Source: https://spectrum.ieee.org/nasa-designed-perseverance-helicopter-rover-fly-autonomously-mars

**Mars Dust Storms — Scale and Duration**
- NASA Earth Observatory: "Dusty Differences Between Mars and Earth"
  - Key finding: Local Mars storm = size of Arizona. Regional = size of USA. Global storms every 3–4 Mars years. Some storms grow, others don't — unpredictable.
  - Source: https://earthobservatory.nasa.gov/images/149926/dusty-differences-between-mars-and-earth

- NASA: "The Fact and Fiction of Martian Dust Storms"
  - Key finding: Solar panels are the primary vulnerability. Dust in atmosphere reduces surface illumination. Global storms can last weeks to months.
  - Source: https://www.nasa.gov/solar-system/the-fact-and-fiction-of-martian-dust-storms/

- Marspedia: "Dust Storms"
  - Key finding: Illumination can drop to 1% at peak storm intensity for up to three months (2001 and 2018 storms). Tau of 4.7 = ~99% opacity.
  - Source: https://marspedia.org/Dust_storms

**Mars Dust Devils**
- Wikipedia: "Martian Dust Devils"
  - Key finding: Martian dust devils up to 8 km tall, 700 m wide, lasting 25+ minutes. Occur at rate of ~1 per sol per km². Beneficial encounters have cleaned rover solar panels.
  - Source: https://en.wikipedia.org/wiki/Martian_dust_devils

**Mars Dust Properties**
- Arizona State University Mars Education: "Dust"
  - Key finding: Mars dust particles 3–20 micrometers, five to ten times finer than talcum powder. Stay suspended for weeks to months. Electrostatic properties cause adhesion.
  - Source: https://marsed.asu.edu/mep/dust

---

## Key Research Findings That Changed My Thinking

### Finding 1: Wind Force Is Not the Danger
Initial assumption was that a dust storm on Mars would be dangerous because of wind. Research showed that Mars atmospheric density (~1% of Earth's) means a 13 km/h storm exerts almost no physical force. The reframe to dust coating, power loss, and electrostatic damage was the most important conceptual shift.

### Finding 2: Storm Classification Is Impossible With Confidence
I initially planned a classification-based approach (classify storm type, then act accordingly). Research into storm behavior showed that even NASA scientists with orbital data cannot predict which storms will grow. A local event can expand to regional scale within hours. This made classification-based decision making fundamentally unreliable — which led to the risk scoring approach instead.

### Finding 3: Ingenuity's Failure Mode Was a Cascade
The Ingenuity communications blackout in May 2022 was caused by dust → reduced solar → power loss → FPGA clock reset → heater shutdown → time desync with rover. This is precisely the cascade I model in the threat section. It confirmed that the threat model is correct and that cascade effects need to be anticipated, not just individual threats.

### Finding 4: Ingenuity Already Had Autonomous Emergency Landing
Ingenuity was upgraded mid-mission to autonomously select landing sites in treacherous terrain. It performed three emergency landings. This proves that autonomous emergency response is achievable and has real Mars precedent — making the proposed solution realistic rather than speculative.

---

## AI-Assisted Research Process

The following is an edited log of the AI conversation that helped structure the problem analysis. Included as permitted by the assignment guidelines.

### Initial Problem Framing

**My question to AI:** How should I approach the problem of a Mars aircraft detecting a dust storm 12 minutes ahead when Earth communication has a 14-minute round trip delay?

**Key AI insight that shaped my thinking:**
> "This is not a navigation problem. It is a classification problem under uncertainty with asymmetric error costs. The aircraft cannot know whether the storm is a dust devil, local event, regional storm, or the beginning of a global event. Each demands a completely different response. The system must decide before the classification is certain."

This framing became the central thesis of Part 1.

### Threat Model Development

**My finding:** A 13 km/h storm on Mars feels like a light breeze on Earth due to atmospheric density differences. Asked AI to help think through what the actual threats are.

**Discussion that followed:** Identified three threat vectors — power loss, sensor blindness, electrostatic damage — and then worked through how they interconnect. The cascade model (dust → solar → power → sensors → decision quality → more dust exposure → electrostatics) emerged from this discussion.

### Classification vs Risk Scoring

**Problem identified:** The original classification-based architecture assumed we could reliably determine storm type. Research showed this was not possible.

**AI suggested reframe:** Rather than asking "what type of storm is this?", ask "what is the current risk level and how fast is it changing?" This led to the four-component risk scoring formula which handles uncertainty explicitly by pricing it into the score via the uncertainty penalty component.

### Graceful Degradation

**Problem identified:** The original architecture did not address what happens when sensors start failing during the storm.

**Discussion:** Worked through sensor failure sequence (optical first, then radar, then onboard-only) and designed the degradation responses. Key insight: the system should never be in a state where sensor failure causes it to take a more aggressive action — it should always degrade toward conservatism.

---

## What I Ruled Out and Why

**ML-based storm classification:** No training data exists. Mars aircraft storm encounter data is essentially zero. ML in safety-critical contexts with insufficient training data produces confident wrong answers, which is worse than acknowledging uncertainty.

**Waiting for clearer sensor data before deciding:** Rejected because of the threat cascade. Waiting for clear evidence of danger means some degradation has already occurred. The decision must precede the evidence becoming unambiguous.

**Contacting Earth for guidance:** Mathematically impossible — Earth's earliest response arrives 2 minutes after the storm contact point. Not a viable option regardless of communication system design.

**Pre-programmed fixed response:** Rejected because the correct response genuinely depends on storm type and aircraft state. A fixed "always land when storm detected" response would abort too many missions unnecessarily. The risk-scoring approach allows proportional response.
