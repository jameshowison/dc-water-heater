# DC Water Heater — Exploration Journal

This journal records the design explorations for a 48V DC water heating system in an RV. Entries are in chronological order. See `plan.md` for current status.

---

## Entry 1 — RV Water System Constraints (early context)

**Problem statement:** The existing RV water heater is a Suburban propane/electric combo tank. Core problems identified:

- Heating a full tank is too power-hungry relative to the small water draws needed (dish washing, hand washing)
- The propane burner design uses a flue tube running through the center of the tank — this acts as a chimney when the burner is off, causing significant standby heat loss
- Tanks need protection against corrosion (anode rod management) and require draining for freeze protection
- Hot water sitting in pipes between the tank and the faucet is wasted on each draw

**Initial requirements:**
- Power source: existing 48V 192Ah RV battery system (can deliver 200A continuous)
- Avoid propane entirely if possible
- Handle low-flow faucet use efficiently
- Freeze-safe design

---

## Entry 2 — Induction Heating Explored (chat: 3d3a96e3)

Investigated induction heating as the primary heating mechanism, motivated by the desire for zero galvanic contact between the 48V supply and the water.

**How it works:** AC current through a coil induces eddy currents in a conductive pipe section (carbon steel or 304SS), which self-heats resistively. The coil itself is not in contact with water.

**First-principles calculation at 0.5 GPM:**
- ΔT target: 25°C → requires ~52W (very modest)
- Skin depth determines how deep current penetrates; higher frequency = shallower

**Solenoid vs pancake coil:**

Solenoid (coil wraps around tube) was compared against a pancake (flat coil over a perforated disc):

- Pancake at 1500W+ requires a disc >100mm diameter to avoid local boiling — defeats compact inline goal
- Solenoid over a 40mm tube gives ~50cm² active area along its length — much more forgiving heat flux
- Pancake feasible at low power (the original 52W concept) but not at shower-scale

**ZVS resonant inverter:** From 48V DC, a resonant LLC or series-resonant inverter (ZVS topology) drives the coil at 20–100kHz. Efficiency ~90–95%. At 3000W input the coil carries 150–300A peak AC, requiring 8–10mm OD copper tube and active water cooling of the coil.

**Two-unit vs single-unit at 3000W:**
- Single 3000W coil: coil current doubles, copper sizing becomes critical, active coil cooling needed
- Two 1500W units: each coil carries half the current, 6mm copper tube air-cooled is sufficient

**Why induction was not pursued further:**

The power electronics complexity (ZVS inverter design, resonant tank tuning, coil winding) is significant. The galvanic isolation benefit can be achieved more simply with a sheathed resistive element (Incoloy sheath isolates the conductor from water). The induction path was set aside in favor of resistive heating — documented here to avoid re-exploring without strong new justification.

---

## Entry 3 — Pressure and Thermal Safety in Triclamp Vessels (chat: 440924ad)

Investigated whether a triclamp spool could serve as a water vessel and what thermal/pressure risks apply.

**Key findings:**

- Standard EPDM-gasketed triclamp fittings: rated ~150 PSI
- Water reaches 150 PSI at ~185°C (365°F) — only reachable in a sealed, heated-under-pressure scenario
- Normal boiling (100°C) in a vented spool: no risk
- **Thermal expansion risk in sealed vessels:** if both inlet and outlet valves are closed and the vessel is heated, even modest temperature rise (20°C → 60°C) in an incompressible fluid can spike hundreds of PSI with no room to expand
- RV water systems typically lack the check valve that creates closed-system problems in residential plumbing — thermal expansion pushes back into the supply line
- A PRV (pressure relief valve) is good practice regardless

**48V 1500W element as thermostat:** Briefly explored whether pressure rise from heating could actuate a switch — not pursued.

---

## Entry 4 — Tube Volume and Flow Heating Calculations (chats: 5383b80a, c6e81b48)

Supporting calculations establishing sizing context.

**2" sanitary tube, 8" long:**
- Bore volume: ~360 mL (≈ 21.96 in³)
- At 0.5 GPM flow: residence time ~11.4 seconds
- Time to heat 360mL by 35°F at 1500W: ~19.5 seconds
- Conclusion: flow-through heating of a 2" tube at 0.5 GPM and 1500W is marginal — the water barely has time to heat

**3" sanitary tube, 8" long:**
- Bore volume: ~847 mL

**1¼" brass pipe, 8" long (two pipes):**
- Liquid volume: ~337 mL total
- Wall metal volume: ~170 mL equivalent

These calculations informed the decision that a flow-through tankless design requires either very high power or very low flow rates to achieve meaningful ΔT. Storing thermal mass (a small tank) is more practical than pure flow-through.

---

## Entry 5 — Temperature Control: Snap Disc and ESP32 Thermostat (chat: f9413753)

Investigated temperature cutoff and control options for the 48V heating element.

**Safety cutoff:**
- Goal: cut power to EV200 contactor coil at 110°F water temp
- Bimetallic snap disc thermostat (NC, opens at 110°F) wired in series with coil signal — simplest approach, no electronics
- Mounting: clamp or strap to copper nipple with thermal paste; self-fusing silicone tape to secure
- Triclamp-compatible temperature switch exists (NPT process side) but requires adapter

**Control thermostat:**
- Desired feature: one-button toggle between normal setpoint (~85°F) and boost setpoint (~104°F)
- ITC-308 has two setpoints but not a clean boost button
- **Selected approach:** ESP32 + NTC probe + relay. One button toggles between two hardcoded temperatures. Simple code, $5 in parts.

**Control electronics stack:**
- 48V bus → Pololu D24V10F5 (48V→5V regulator) → ESP32
- ESP32 GPIO → 5V single-channel relay module (SRD-05VDC-SL-C based)
- Relay contacts switch 12V coil signal to EV200 (contacts rated 30VDC/10A — trivial load)
- **PID vs PWM:** PID calculates output based on error, rate of change, and accumulated error — appropriate for a contactor (on/off output with smart timing). PWM varies duty cycle at fixed frequency — appropriate for direct power modulation, not for a contactor.
- Wide hysteresis needed to avoid rapid contactor cycling (EV200 not rated for high cycle counts)

---

## Entry 6 — DC Tankless Heater: Rheem Vessel Strategy (chat: baaf5cff)

Explored using a commercial tankless water heater vessel as the heating chamber.

**The borked Rheem RTEX strategy:**

Rheem RTEX-18 (18kW, 240VAC) has a well-known failure mode: control board dies, copper vessel and elements survive. Dead units appear on Facebook Marketplace/Craigslist for $30–50.

- RTEX-13/18: 2 copper flow tubes, manifolds, flow switch, thermistors, thermal cut-off
- RTEX-27: 3 tubes — more interesting for staged power control

**Element compatibility:**
- Existing elements: HE90240, 9kW/240V, 1" NPSM thread, ~6.4Ω
- At 48V: 48²/6.4 = 360W — too low as-is
- **DERNORD 48V/1500W element:** 1" NPSM thread (confirmed compatible), ~1.54Ω, delivers 1500W at 48V
- Drop-in replacement: pull HE90240, install DERNORD, rewire to ESP32 + MOSFET

**Multi-leg contactor architecture:**

With 3× DERNORD at 48V (one per tube, one contactor per element):
- Each contactor switches 31A — P115 at 62% of 50A rating, comfortable
- Staged power: 1.5kW / 3.0kW / 4.5kW
- At 0.5 GPM: 1.5kW → +41°F, 3kW → +82°F, 4.5kW → +123°F (TMV needed above 1 element)
- ESP32 staging logic: flow detected → element 1 on → if below target → element 2 → etc.
- TMV (thermostatic mixing valve) handles fine temperature control; contactors handle power steps

**Contactor selection:**
- EV200 (Gigavac/Tyco): 12V coil variant, 48V/500A rated, fast (10ms), built-in coil suppression — no external flyback diode needed
- P115BDA (12V coil) / P115FDA (48V coil): 50A rated, quieter than EV200, also have built-in suppression
- P115FDA/BDA: FDA = 48V coil, BDA = 12V coil (needs small buck converter but more available used)
- **On hand:** EV200 and P115 contactors

**96V boost path explored and rejected:**

To use commercial 240VAC tankless heaters from a 48V bus, a 48V→240VDC boost converter was considered. Rejected: the ease and cost-effectiveness of storing heat in water (thermal mass) makes the boost converter complexity unjustifiable. A well-designed insulated tank outperforms the tankless approach for RV use patterns.

**Nichrome rewind explored:**

Investigated rewinding the HE90240 element with nichrome 80 wire to target 1.54Ω at 48V.
- Target wire: 18 AWG nichrome 80 (1.02mm), ~1.346 Ω/m
- Two coils of ~1.977m each in parallel → ~1.33Ω → 1,734W at 48V
- Feasible but custom elements from Tempco/OEM Heaters (~$40–60) are more practical

**RTEX vessel outcome:**

Physical test revealed the RTEX copper flow tubes are much smaller than expected (~1¼" diameter). This gives too little water volume per tube — insufficient thermal mass for the RV use case. The RTEX vessel strategy was abandoned. [photo: chat baaf5cff]

---

## Entry 7 — Faucet-Integrated Heater Concept (chat: 7fb9973b)

Stepped back to reframe the problem. Core issues restated:

- Heating a full 6-gallon tank for small faucet draws wastes energy
- Water sitting in pipes between tank and faucet is wasted (cold water purge before hot arrives)
- The existing propane/electric Suburban tank has inherently poor insulation (flue chimney effect)

**Design direction:** point-of-use heating at the faucet, with a pre-tempered supply from an upstream tank. Eliminates cold water purge, reduces tank heating load.

**Heating element selection for faucet body:**

Options evaluated:
- Resistive (sheathed element): simplest, well-understood, but 48V DC in water contact raises shock concern
- PTC (positive temperature coefficient ceramic): self-limiting temperature, no thermostat required, electrically isolated if potted — but 48V DC rated PTC elements at 5kW are non-trivial to source
- Induction: zero galvanic contact with water, no element corrosion, fast response — adds DC→AC inverter stage (see Entry 2)
- **Selected: induction initially, then revised to sheathed resistive** — Incoloy sheath provides adequate isolation, simpler electronics

**Physical layout question:** Induction driver hardware at the faucet vs under-sink. At 5kW the driver generates heat and needs clearance — separating heater unit (under-sink) from faucet valve is more practical.

---

## Entry 8 — Faucet Heater with Incoloy Element, Full Design (chat: 990abbdd)

Most evolved design session. Consolidated the faucet heater and upstream tank concepts.

**Two-stage architecture confirmed:**
1. **Upstream tank** (under counter): pre-tempered water store, Dernord 48V/1500W element, insulated
2. **Faucet body heater**: in-line boost at point of use, feeds from tank

**Faucet body spec:**
- Body: 2" stainless sanitary tube, ~14" long, ~840mL volume
- Element: DERNORD 48V/1500W, U-bend, 1" NPSM, terminals at bottom
- Insulation: Armaflex wrap on body
- Mounting: vertical, deck-mount under counter
- Control: ESP32, PIR motion sensor (predictive pre-heating), flow switch (activation gate), NTC thermistor at outlet
- Flow gate: NC solenoid, temperature-gated by ESP32 (no flow if element not ready)
- Handle: lever microswitch

**Contactor flyback:**
- EV200: built-in coil suppression, no external flyback diode needed
- P115BDA/FDA: also built-in suppression, confirmed for both variants
- Sharing 12V rail between contactor coil and ESP32: safe with either unit; add decoupling cap on ESP32 power input as good practice

**Tank design evolution:**

The Suburban propane tank was evaluated for conversion:
- Flue tube penetrations (top and bottom) require welding to plug — feasible but adds a fabrication step
- **Fogatti HybridShower 6 Ultra** identified as best drop-in commercial option: fits 13×13" cutout, EPDM foam insulation (<25% heat loss in 24 hours), dual fuel but gas line simply not connected, 1440W 120VAC element → swap for DERNORD 48V/1500W, ~$300
- **Custom SS tank** also designed: 6-gallon SS304 pressure vessel, 1" NPSM element port, ½" NPT drain/PRV/inlet/outlet, 1" aerogel blanket + 2" closed cell spray foam, estimated standby loss ~4–5W

**Super-insulated tank thermal performance:**

| Insulation | Standby loss | Tank at 9pm (heated to 110°F at 3pm) |
|---|---|---|
| Suburban (propane flue) | ~71W | ~85°F |
| Fogatti (EPDM foam) | ~25% / 24hr | ~107°F |
| Custom aerogel+foam | ~4–5W | ~109°F |

**Propane generator as backup:** Rather than propane water heating, a small propane inverter generator (Honda EU2200i or similar, ~1.8kW) used on cloudy days charges the 48V battery, which then runs everything including the water heater through the same DC path. Eliminates dedicated propane water heating plumbing entirely.

**Current tank plan:** Custom triclamp vessel using stainless sanitary tube/spool components with DERNORD element. Exact dimensions TBD pending offline design work. [photo: chat 990abbdd]

**MOSFET gate drive open question:**

ESP32 GPIO is 3.3V; IRFP4568 Vgs(th) ~4V — marginal enhancement. May need a gate driver IC between ESP32 and MOSFET gate. Not yet resolved.

**BoM document created** during this session: `dc-faucet-heater-bom.md` (partial, to be updated).

---

## Entry 9 — Control Circuit Detail and Open Questions (chat: 335e1db9)

Produced a detailed BoM and circuit open questions document.

**Control architecture:**
- 48V bus → ANL fuse (80A) → MOSFET (IRFP4568 or equiv) → DERNORD elements
- EV200 contactor as master disconnect (safety interlock)
- Flow switch opens EV200 coil circuit on flow stop — hard disconnect independent of ESP32 state
- 48V → 12V buck (Pololu or similar) powers ESP32 and contactor coil
- NTC thermistors (10kΩ @ 25°C, B=3950) at inlet and outlet

**Open questions documented (for future resolution):**

*Electrical:*
- Measured resistance of DERNORD 48V/1500W at operating temperature (cold R = 1.54Ω)
- Two elements in parallel = 0.77Ω → 3000W at 48V — sufficient for 0.5 GPM worst-case inlet?
- ESP32 3.3V GPIO → IRFP4568 gate: fully enhanced? Gate driver IC needed?
- Flyback diode across MOSFET for inductive transients from wiring? (Resistive load — probably not needed)

*Plumbing/vessel:*
- DERNORD 1500W physical fit in RTE-13 brass tubes — confirmed thread compatible, water tightness not tested, tubes too small (abandoned)
- Outlet thermistor placement: tee on hot outlet pipe vs boss on vessel

*Potable water safety:*
- RTE-13 brass vessel alloys — lead content in older brass? (Moot — RTEX vessel abandoned)
- DERNORD stainless SUS304 — adequate for potable water, or specify 316L?
- Electrical isolation: is sheathed element inherently isolated from water, or is bonding/grounding needed?
- Electrolysis risk at 48V DC with sheathed elements?
- Copper pipe vessel pressure rating at 60–80 PSI mains?

*Control:*
- PID thermal lag: time between element power change and outlet thermistor response at 0.5 GPM
- Minimum flow rate before activation (flow switch threshold)
- Behavior at very low flow (dribble) — element overheat before flow switch activates?

---

*Journal continues as design progresses. See `plan.md` for current state.*
