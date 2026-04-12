# DC Water Heater — Exploration Journal

This journal records the design explorations for a 48V DC water heating system in an RV, in chronological order. See `plan.md` for current status.

**RV power system context (fixed throughout):** 48V, 192Ah LiFePO4 battery bank, 200A continuous delivery.

---

## Entry 1 — Problem Statement and Initial Requirements (background context)

The existing RV water heater is a Suburban propane/electric combo tank. Core problems identified:

- Heating a full tank wastes energy relative to the small water draws needed for dish washing and hand washing
- The existing tank loses heat faster than expected — likely poor insulation; the propane burner flue running through the center of the tank may act as a chimney when off, but this is unconfirmed
- Tanks need protection against corrosion (anode rod management) and require draining for freeze protection
- Hot water sitting in pipes between the tank and the faucet is wasted (cold water purge before hot arrives)

**Initial goal:** Replace the propane/electric system with a 48V DC system powered by the existing RV battery, optimized for low-flow faucet use.

---

## Entry 2 — Faucet Heater Concept, Flow Geometries, and Tank Buffer (2026-03-21, chat: 7fb9973b)

First serious design session. Focused on a point-of-use faucet heater fed from a 48V DC source.

**Power requirement at real flow rates:**

Early confusion between 0.5 g/s (a drip) and 0.5 GPM (a real faucet). Corrected numbers:
- Flow: 1.5 GPM = 95 ml/s
- ΔT = 35°C (cold inlet to usable temperature)
- Power needed: ~14,000W

This is why commercial under-sink units are 7–18kW.

**Heating element options evaluated:**
- Resistive sheathed (Incoloy): simplest, but 48V DC in water contact raises shock/electrolysis concern
- PTC ceramic: self-limiting temperature, no thermostat needed, isolated if potted — but 48V DC rated at 5kW hard to source
- Induction: zero galvanic contact with water, no element corrosion — adds DC→AC inverter stage
- **Initial selection: induction** (see Entry 5 for full induction exploration, subsequently rejected)

**Flow geometry analysis:**

Three geometries evaluated:

- *Single-pass straight tube:* simple, cheap, hot spots where element is hottest, scaling concentrates there
- *Coiled tube:* longer water path, more contact time, better turbulence, harder to drain
- *Multi-pass / U-tube or helical:* most efficient heat transfer, highest pressure drop, hardest to drain

Key insight: freeze drainability vs heat transfer efficiency are in direct opposition. Initially prioritized drainability for RV use — then confirmed that compressed air blow-out makes drainability a non-constraint, freeing design to optimize for heat transfer.

With drainability removed: **helical/coiled geometry wins.** At low flow rates (laminar regime), coiled geometry induces **Dean vortices** — secondary flow patterns that continuously mix hot wall water with cool center water, dramatically improving wall-to-water heat transfer without increasing pressure drop.

**Element surface temperature analysis:**

At 5kW with a typical element surface area of ~0.02m²:
- Heat flux q ≈ 250,000 W/m²
- With Dean-vortex-enhanced convection: element surface ~71°F above local water temp
- At outlet (136°F water): element surface ~207°F — approaching boiling, risky at low flow

Conclusion: 5kW into a small coil is aggressive. Either lower watt density (larger element surface) or ensure flow never drops below ~1 L/min while energized.

**Tank-as-buffer decision:**

A larger diameter pipe section (~1.5L) as a small thermal buffer simultaneously:
- Reduces watt density (same element, more water volume, lower surface temp)
- Provides thermal mass for burst demand
- Makes behavior more forgiving at low flow

**TMV (thermostatic mixing valve):**

At high outlet temperatures, scald risk is real. A TMV with a hard cap at 49°C specified — Honeywell AM101 or Watts LF1170, ~$35–40. This also simplifies control: ESP32 just manages flow switch; TMV handles fine temperature.

**Status at end of session:** Induction selected pending deeper investigation. Tank-as-buffer architecture established. TMV specified. Flow geometries resolved.

---

## Entry 3 — Rules of Thumb for Electric Water Heating (2026-03-22, chat: c8fa0155)

A dedicated session working out the physics of electric water heating from first principles, motivated by two things: annoyance at the "you need 5 gallons to wash dishes" claim in RV forums, and the question of whether the battery-as-buffer concept from an impulse cooktop could be extended to hot water.

**Key findings:**

Resistance heating is 100% efficient — every watt becomes heat, making the math unusually clean.

**Energy rules of thumb (US customary):**
- 120 Wh/gal summer (70°F inlet → 120°F setpoint)
- 200 Wh/gal winter (40°F inlet → 120°F setpoint)
- Winter ≈ 1.5× summer (not 2×)
- At 160°F setpoint: 300 Wh/gal — 50% more stored energy, same tank

**Actual hot water volumes by activity:**

| Activity | Gallons (hot) |
|---|---|
| Hand wash (hands) | 0.5 |
| Dish basin | 5 |
| Short shower (5 min, low-flow) | 5 |
| Long shower (10 min) | 15 |

A short shower and a dish basin are the same volume. Both are ~0.6–1.0 kWh depending on season.

**Tankless power formula:** P (W) = flow (L/min) × ΔT (°C) × 69.8

**The voltage/current wall:**

At the sink in winter (2 L/min, ΔT = 45°C):

| Voltage | Amps |
|---|---|
| 52V | 121A |
| 120V | 52A |
| 240V | 26A |

**Mnemonic:** at 120V → ~50A per sink winter; at 52V → ~120A per sink winter. The system voltage and the amp draw swap. Shower = 5× sink flow = 5× amps. Tankless shower at 52V or 120V is not practical. **Tankless shower requires 240V or gas, full stop.**

**Why low voltage doesn't work for high-power tankless:**

A 4500W/52V element needs R = 0.6Ω. A 4500W/240V element has R = 12.8Ω. Getting 0.6Ω in the same tube requires ~20× larger wire cross-section — 4–5× the diameter — which won't coil into the element tube. Physical impossibility, not a design tradeoff.

**Tank recharge at 52V / 1500W:**
- ~14 minutes per shower-worth of water (from a warm tank)
- ~60 minutes full cold recharge of 5 gallons

**Conclusion:** The impulse cooktop buffer concept applies — charge slowly, deliver fast — but water tolerates a much gentler arrangement than cooking. A 5-gallon tank at 160°F stores ~1,500 Wh, enough for a full low-flow shower. Your existing tank, rewired to the DC bus with a DERNORD element, is already the answer.

**Blog post produced:** `water-heating-offgrid.md` — long-form Hackaday-style post covering the full journey from irritation to rules of thumb. Tone: conversational, DIY/off-grid audience.

---

## Entry 4 — DC Tankless Heater: Elex Teardown and Rheem Vessel Strategy (2026-03-23, chat: baaf5cff)

**Elex 5.5 kW unit:**

The Elex 5.5 uses bare nichrome coils floating in water-filled copper bores (11mm ID bore, coil OD ~8.3mm, coil ID 7.5mm, wire ~0.4mm diameter, openly wound at ~4.25mm pitch). Water flows inside and outside the coil — good heat transfer geometry. Elements terminate through ceramic bead insulators into screw terminals.

Resistance of each element: ~8.7Ω. At 48V: 48²/8.7 = **265W per element** — far too low.

**Rewind analysis:**

To hit 1.9kW per element at 48V: need R = 1.21Ω. With bore length 170mm and coil ID 7.5mm, 18 AWG (1.02mm) nichrome 80 wire fits and gives the right resistance. Two coils in parallel: ~0.77Ω → ~3000W at 48V. Physically feasible. [photo: chat baaf5cff]

**Borked Rheem RTEX strategy:**

Rheem RTEX units (18kW, 240VAC) fail via board death, leaving copper vessel intact. Dead units on Marketplace for $30–50. DERNORD 48V/1500W uses the same **1" NPSM thread** as the HE90240 stock element — confirmed drop-in compatible (NPSM is straight-thread, seals via face gasket, compatible with NPT female fittings). At 48V per DERNORD: 1500W, 31A.

RTEX models: 13kW (2 tubes), 18kW (2 tubes), 27kW (3 tubes). 3× DERNORD 1500W in RTEX-27 = 4500W staged.

**Multi-leg contactor architecture:**

One P115 contactor per element at 31A = 62% of 50A rating. Staged power: 1500W / 3000W / 4500W. TMV handles fine temperature; contactors handle power steps. Both EV200 and P115 have built-in coil suppression — no external flyback diode needed.

**Contactor options:**

| Contactor | Coil | Rated | Notes |
|---|---|---|---|
| EV200 | 12V | 48V/500A | Fast, built-in suppression, loud |
| P115BDA | 12V | 50A | Quieter, built-in suppression, common used |
| P115FDA | 48V | 50A | Same but 48V coil, less common used |

**96V boost path explored and rejected:**

To run stock 240VAC elements from 48V bus, boost converters were investigated. At 8.8kW: input ~183A from 48V bus; boost converter ~$300–500. **Rejected:** thermal storage (tank) is cheaper and simpler. A well-insulated tank outperforms tankless for RV use patterns.

**Corrosion analysis for copper vessel:**

- Galvanic: stainless sheath (anode) in copper pipe (cathode) — mild with clean water; mitigated by dielectric union on inlet/outlet
- DC leakage from sheath to water: test sheath-to-terminal resistance periodically; should read infinite
- Scale: periodic citric acid flush
- No anode rod needed — no steel in the system

**RTEX vessel outcome:**

Physical inspection revealed RTEX copper flow tubes are ~1¼" diameter — too small, insufficient water volume. RTEX vessel strategy **abandoned**. [photo: chat baaf5cff]

**Status at end of session:** Rheem vessel path closed. DERNORD 1500W element confirmed as correct part. Multi-leg contactor architecture established. 96V boost rejected. Rewind path still open pending electrolysis investigation.

---

## Entry 5 — DC Electrolysis Safety, Rewind Hard Stop, and Control Circuit BoM (2026-03-24, chat: 335e1db9)

**Nichrome rewind risks:**

While investigating rewinding the Elex nichrome, bare-wire-in-water under DC was explicitly questioned for drinking water use.

Key findings:
- AC self-cancels electrolysis effects each half-cycle; DC applies constant polarity → sustained electrolysis
- Nichrome 80 forms a chromium oxide passive layer offering some protection, but DC continuously attacks it
- Every 10°F temperature rise doubles corrosion rate
- **For drinking water / dishwashing: hard stop.** Nickel and chromium ions are toxic at elevated concentrations. DC will continuously drive dissolution regardless of oxide layer.

The Elex 5.5 bare nichrome bore geometry is also incompatible with a sheathed element — bores are too narrow. Elex rewind path **closed**.

**Pivot to sheathed elements:** Incoloy sheathed element (metal tube around nichrome, no water contact with energized conductor) eliminates the AC vs DC concern entirely. Custom 48V Incoloy-sheathed elements available from OEM Heaters (oemheaters.com). DIY copper pipe flow housing as vessel — standard plumbing tee with screw-plug element.

**Control architecture documented:**

```
48V bus → [ANL fuse 80A] → [EV200 contactor] → [MOSFET IRFP4568] → DERNORD elements
48V bus → [48→12V buck] → ESP32 + contactor coil
Flow switch in series with EV200 coil → hard disconnect on flow stop, independent of ESP32
NTC thermistors (10kΩ @ 25°C, B=3950, M4 probe) at inlet and outlet
```

**Open questions documented:**

*Electrical:*
- DERNORD 48V/1500W cold R = 1.54Ω; two in parallel = 0.77Ω → 3000W — confirm measured resistance at operating temp
- ESP32 GPIO is 3.3V; IRFP4568 Vgs(th) ~4V — may need gate driver IC
- Flyback diode across MOSFET: resistive load, probably not needed, confirm

*Plumbing / potable water:*
- Outlet thermistor placement
- DERNORD sheath material SUS304 vs 316L for potable water — specify 316L
- Electrical isolation of sheathed element from water — bonding/grounding requirements
- Electrolysis risk at 48V DC with sheathed elements

*Control:*
- PID thermal lag at 0.5 GPM
- Minimum flow threshold for flow switch — prevent dry-fire at dribble flow
- Stripboard layout not yet produced

**Status at end of session:** Bare wire path definitively closed. Sheathed element confirmed as only viable path for potable water. Control architecture drafted. MOSFETs not yet on hand.

---

## Entry 6 — In-Faucet Heater with Incoloy Element: Full Design (2026-03-31, chat: 990abbdd)

**Two-stage architecture confirmed:**

1. **Upstream tank** (under counter): pre-tempered water store, Dernord element, insulated
2. **Faucet body heater**: in-line boost at point of use, feeds from tank

**Faucet body design:**

- Body: 2" stainless sanitary tube, ~14" long, ~840mL volume, Armaflex insulation
- Element: DERNORD 48V/1500W, U-bend, 1" NPSM, Incoloy sheath, terminals at bottom
- Control: ESP32, PIR motion sensor (predictive pre-heating), flow switch, NTC thermistor at outlet
- Flow gate: NC solenoid, temperature-gated (no flow until water is at temperature)
- Handle: lever microswitch
- Mounting: vertical under counter

**Aluminum block vs pipe + Dernord:**

Evaluated aluminum block with cartridge heaters as alternative vessel. For the boost stage specifically (flow-through by nature — pre-tempered water needs 20–25°F lift) the aluminum block argument partially applies. However, Dernord pipe body wins on simplicity and potable water safety; boost logic belongs in control system, not a separate vessel.

**Upstream tank options evaluated:**

- *Fogatti HybridShower 6 Ultra:* best drop-in commercial option (13×13" cutout, EPDM foam, <25% heat loss/24hr, 1440W 120VAC element → swap for DERNORD 48V/1500W, ~$300)
- *Custom super-insulated tank:* aerogel + closed cell foam, ~4–5W standby loss — subsequently superseded by triclamp vessel approach
- *Custom triclamp vessel:* current plan (see Entry 7)

**Propane generator as backup:** On cloudy days, a small propane inverter generator charges the 48V battery → same DC path serves all heating. No dedicated propane water heater plumbing needed.

**DERNORD 1500W element fit confirmed:** R = 1.54Ω cold → 1500W at 48V. Ordered.

**Sharing the project:** Decided on Hackaday.io for project logs + GitHub repo for ESP32 firmware and BoM. No Hackaday draft written yet.

**Status at end of session:** Two-stage architecture locked. Faucet body spec complete. DERNORD element ordered. Tank approach still in flux.

---

## Entry 7 — Dernord Element Thread and Copper Pipe Vessel (2026-03-31, chat: 5343810f)

**Thread clarification:**

DERNORD 48V/1500W is 1" NPSM — a straight thread that seals via face gasket. Confirmed compatible with standard 1" NPT female fittings — face gasket seals regardless of thread taper mismatch.

**Copper pipe vessel exploration:**

Explored 2" copper pipe as tank body with a soldered 1" NPT weld bung for the element port and SharkBite push-fit ½" side inlet. Pursued briefly for prototyping simplicity, then **abandoned in favor of stainless triclamp** (Entry 7) due to:
- Copper's high thermal conductivity increases standby heat loss vs stainless
- Soldering/brazing required for NPT bung — not tool-free
- Triclamp offers cleaner, fully disassemblable construction

---

## Entry 8 — Triclamp Vessel: Pivot from Copper (2026-04-01, chat: 897332ec)

**Why triclamp:**

- Stainless steel thermal conductivity ~15 W/m·K vs copper ~400 W/m·K → significantly lower standby heat loss
- Fully disassemblable — no soldering, clean serviceability
- DERNORD 2" tri-clamp × 1" FNPT adapter exists off-shelf — element port solved without fabrication
- All-clamp construction handles RV vibration well with EPDM gaskets + hex bolt clamps

**Vibration:** Mount tube rigidly; EPDM or silicone gaskets (more compliant than PTFE); hex bolt clamps preferred over wing nut.

**Temperature sensing:** ½" NPT side ports in SS spool tube wall for direct water temperature measurement. DERNORD sells spool pieces with NPT side ports.

**Triclamp BOM (draft):**

| Component | Spec |
|---|---|
| Vessel body | 2" SS304 tri-clamp spool tube, ~14" |
| Element port (right end) | DERNORD 2" TC × 1" FNPT adapter |
| Left end cap | 2" TC blank ferrule |
| Inlet (bottom-left) | 2" TC spool with ½" NPT side port, PEX adapter |
| Outlet (top-right) | 2" TC spool with ½" NPT side port, 3/8" faucet fitting |
| Temp ports | ½" NPT thermowell in spool side ports |
| Clamps | 4× 2" hex bolt TC clamps |
| Gaskets | 4× 2" EPDM |
| Insulation | 2" foam pipe sleeve + end foam |
| Element | DERNORD 48V/1500W, 1" NPSM |

**Suppliers:** McMaster-Carr (best documentation), Glacier Tanks / BrewHardware.com (brewing-focused, good 2" selection), Beduan / QiiMii on Amazon (good Dernord alternatives). Copper pipe vessel path **abandoned**.

**Status at end of session:** Triclamp SS vessel confirmed as tank architecture. BOM drafted. Offline design work to follow.

---

## Entry 9 — DC Power Safety Research: Confirmation of Hard Stop (2026-04-01, chat: 1c02e9c2)

Follow-up research session confirming the bare wire DC electrolysis conclusion from Entry 4.

**Research findings:**

- Trade knowledge confirms DC as the primary culprit for electrolytic corrosion; AC needs to be converted to DC before causing metallic corrosion
- **Chinese patent CN102878666B:** specifically an anti-electrolysis design for bare wire heaters, acknowledging electrolytic corrosion and "electrolytic pollution" as real engineering problems. Solution: nickel alloy for all water-contact surfaces with matched electrode potentials, keeping surface electrostatic double-layer potential below 0.8V. This patent confirms the problem is serious enough to warrant dedicated engineering.
- DIY solar community confirms sheathed elements eliminate the AC vs DC concern entirely
- Gap in literature: no rigorous study measuring actual metal ion contamination from DC bare wire heaters in drinking water, but the mechanism is uncontested

**Conclusion:** Prior hard stop confirmed. Sheathed (Incoloy) element is the only viable path for potable water at 48V DC.

---

## Entry 10 — Induction Heating Deep Dive (2026-04-06, chat: 3d3a96e3)

Full investigation of induction heating, which had been selected in Entry 2 but not yet explored in depth.

**How it works:** AC current through a coil induces eddy currents in a conductive pipe section (carbon steel or 304SS), which self-heats resistively. The coil is not in contact with water. Skin depth δ = √(2ρ/ωμ) — higher frequency concentrates currents at surface.

**Solenoid vs pancake coil:**

- *Solenoid:* coil wraps around the tube, heats cylindrical wall. Over a 40mm tube gains ~50cm² active area along its length. Efficient coupling.
- *Pancake:* flat spiral coil, generates field axially through a disc susceptor. The "coin in the tube" geometry — stainless disc, face perpendicular to flow, holes punched through it, pancake coil around the outside of the spout.
  - Interesting for: zero thermal lag, heating right at the tip, large surface area relative to volume
  - Problem at 1500W+: requires disc >100mm diameter to avoid local boiling — defeats compact inline goal
  - Fine at low power (~52W), not viable at shower-scale

**ZVS resonant inverter:** From 48V DC, a resonant LLC or series-resonant inverter drives the coil at 20–100kHz. Efficiency ~90–95%. At 3000W the coil carries 150–300A peak AC, requiring 8–10mm OD copper tube and active water cooling.

**Aluminum block with cartridge heaters (also evaluated):**

At low power, an aluminum block with cartridge heaters in bored holes and a machined water channel provides the same zero-galvanic-contact benefit at much lower complexity — simpler, cheaper, more reliable, and easier to seal than induction.

**Why induction was not pursued:**

The power electronics complexity (ZVS inverter design, resonant tank tuning, coil winding, coil cooling at >1kW) is significant. The galvanic isolation benefit is achieved more simply with a sheathed resistive element (Incoloy sheath isolates the conductor from water). The induction exploration was also superseded by the confirmation in Entries 4 and 8 that sheathed elements are the correct path for potable water. **Documented here to avoid re-exploring without strong new justification.**

---

## Entry 11 — Tube Volume and Flow Heating Calculations (2026-04-08, chats: 5383b80a and c6e81b48)

Supporting sizing calculations:

- 2" sanitary tube, 8" long: ~360mL bore; at 0.5 GPM flow, residence time ~11.4s; 1500W heats 360mL by 35°F in ~19.5s — marginal for flow-through
- 3" sanitary tube, 8" long: ~847mL bore
- 1¼" brass pipe, 8" long (×2): liquid volume ~337mL

**Confirmed:** pure flow-through tankless at RV faucet flow rates requires very high power or very low flow. Tank-as-buffer is the right architecture.

---

## Entry 12 — Pressure and Thermal Safety in Triclamp Vessels (2026-04-12, chat: 440924ad)

Standard EPDM-gasketed triclamp fittings rated ~150 PSI. Water reaches 150 PSI only at ~185°C — not achievable in normal operation. RV water systems typically lack the check valve that creates closed-system thermal expansion problems in residential plumbing; thermal expansion pushes back into the supply line. PRV added as good practice.

Thermal expansion risk if both valves are closed and vessel is heated (sealed incompressible fluid): even 20°C→60°C can spike hundreds of PSI. Mitigated by open RV system and PRV.

At RV supply pressure (~60–80 PSI): well below the 150 PSI EPDM limit. No pressure safety concern under normal operation.

---

## Entry 13 — Temperature Control: Snap Disc and ESP32 Thermostat (2026-04-12, chat: f9413753)

**Safety cutoff:** Bimetallic snap disc thermostat (NC, opens at 110°F) wired in series with contactor coil signal line. Clamped to copper nipple with thermal paste + self-fusing silicone tape. Hardware-level cutoff, no electronics required.

**ESP32 thermostat with boost mode:** ESP32 + NTC probe (10kΩ @ 25°C, B=3950) + 5V relay module. One button toggles between two hardcoded setpoints (~85°F normal, ~104°F boost). 48V → Pololu D24V10F5 (5V) → ESP32. Relay contacts switch 12V coil signal to EV200.

**PID vs PWM:** PID appropriate for contactor (on/off output with smart timing, accounts for error, rate of change, accumulated error). PWM (fixed frequency, variable duty cycle) appropriate for direct power modulation, not for a contactor. Wide hysteresis essential to avoid rapid cycling (EV200 not rated for high cycle counts).

---

## Entry 14 — Faucet Body Heater: Concept Parked (2026-04-12)

**The concept:** A second heating stage at the point of use — a 2" SS sanitary tube body ~14" long (~840mL), DERNORD 48V/1500W element, Armaflex insulated, mounted vertically under the counter. Control via PIR motion sensor for predictive pre-heating, lever microswitch on faucet handle, NTC thermistor at outlet, and an NC solenoid gate held open only when water is at temperature. The idea was to eliminate any cold-water delay at the faucet and provide a boost stage fed from the upstream triclamp tank.

**Decision:** Parked. The plan now covers a single-stage system: triclamp tank only. The faucet body heater adds significant complexity (additional element, solenoid gate, PIR, lever microswitch, second thermistor loop) before the simpler single-stage system has been built and validated. The concept is preserved here for future consideration once the tank stage is working.

---

## Entry 15 — Control Simplification: Thermostat Not FET/PWM (2026-04-12)

**What was considered:** MOSFET (IRFP4568, TO-247) in the 48V power path for fine power control — either PWM duty-cycle modulation of element power or as a solid-state switch driven by ESP32 GPIO. Raised an immediate problem: ESP32 GPIO outputs 3.3V; IRFP4568 Vgs(th) is ~4V, so the gate would not fully enhance without a gate driver IC. This added a component and a design question.

**Decision:** Remove the FET entirely. The ESP32 acts as a configurable thermostat — reads an NTC outlet thermistor, drives the P115 contactor coil(s) via a 5V relay module to switch element power on/off. Two hardcoded setpoints: ~80°F (maintain) and ~104°F (boost), toggled by a single button.

**Why this is sufficient:** The load is purely resistive. PWM on a resistive element would only average power over a cycle — the same effect achieved more simply by the thermostat's on/off duty cycle over longer timescales. The P115 contactors already provide staged power (1× or 2× elements = 1500W or 3000W); thermostat hysteresis controls temperature within a stage. No FET, no gate driver, no PWM frequency to choose.

**What moves out of BoM:** IRFP4568 MOSFET (TO-247), gate driver IC.

---

## Entry 16 — TMV Removed (2026-04-12)

TMV (thermostatic mixing valve) was specified in Entry 2 for scald protection (hard cap at 49°C/120°F) and fine temperature control. With max setpoint now fixed at ~104°F (~40°C) — well below the 49°C scald threshold — there is no scald risk and no need for downstream mixing. Removed from design.

---

## Entry 17 — Flow Switch Removed; Contactor Simplified to P115 per Element (2026-04-12)

**Flow switch removed:** The flow switch was originally wired in series with the EV200 coil as a hard dry-fire interlock — flow stops, element disconnects instantly. This made sense for an inline heater where the element is in a tube and flow is what keeps it cool. The current design is a small pressurized tank: the element is always submerged in standing water. Stopping the faucet doesn't endanger the element; the tank stays full. Thermostat-based temperature control is the appropriate trigger for a tank, not flow detection.

**EV200 removed from circuit:** The EV200's role was specifically the flow-switch-triggered master disconnect. With the flow switch gone, there is no remaining function that justifies the EV200 in the power path. It is oversized (500A rated vs 31A load) and was never intended for thermostat cycling duty.

**Architecture settled:** one P115BDA contactor per element, driven by ESP32 via relay module. P115 is rated 50A — 62% derating at 31A per element, appropriate for switching duty. One or two elements may be fitted; each gets its own P115. ANL fuse remains upstream for overcurrent protection; snap disc in series with coil signal for hardware thermal cutoff.

---

## Entry 18 — Control Power Supply: 48V→12V Architecture (2026-04-12)

**Decision:** Single 48V→12V buck converter powers both the ESP32 and the P115 contactor coils. No separate 5V supply needed.

**Rationale:**

- P115BDA coil is 12V — 12V rail drives the coils directly (via relay module).
- Most ESP32 dev boards accept 7–12V on the VIN pin through their onboard regulator chain (12V → 5V → 3.3V). Dissipates ~1–2W as heat; manageable. Confirm against specific board.
- Relay module voltage changed from 5V (SRD-05VDC-SL-C) to 12V (SRD-12VDC-SL-C) to match the single 12V rail.

**Relay module and GPIO drive:**

The bare SRD-12VDC-SL-C relay coil draws ~70–90mA — too much for direct ESP32 GPIO drive (safe limit ~12mA). The relay MODULE board solves this: it includes an NPN transistor and base resistor; ESP32 GPIO sinks a few mA into the transistor base, which then drives the 12V coil from the supply rail. Signal input threshold is 2–3V, so 3.3V GPIO works cleanly. Buy the module board, not the bare relay.

**Noise:**

Sharing a supply between the ESP32 and contactor coils is acceptable here. P115 has built-in coil suppression (Entry 4). The relay module transistor stage means ESP32 GPIO only switches the low-current relay coil, not the P115 coil directly — two-stage isolation keeps the ESP32 clean. A 100µF cap on the 12V rail near the ESP32 is the first mitigation if glitches appear in testing. Separate supplies not needed for this application.

**Buck converter input voltage:**

Must be rated for full-charge battery voltage, not nominal 48V. A 16S LiFePO4 bank charges to ~58V. Pololu D36V6Fx parts (36V max) and D50V50Fx parts (50V max) are insufficient or marginal. Pololu D57V45F12 (57V max input, 12V output) is the right size class. Confirm input ceiling against actual battery spec before ordering.

---

## Entry 19 — Plan Review and Gap Analysis (2026-04-12)

Full review of plan.md against the design history in explore.md. Items identified and resolved or added as open tasks:

**Resolved in this session:**

- **Outlet thermistor placement:** decided to surface-mount NTC on copper outlet nipple rather than use a thermowell. Thermowell removed from BOM. Copper nipples (½" NPT × ~1") added to inlet and outlet side ports as mounting points for both the NTC thermistor and the snap disc — this also solves the snap disc mounting question from Entry 13 (snap disc needs a copper surface; the copper nipple provides it).

- **Cabling:** documented wire gauge for the ~2ft run from the Victron distribution bar. At 2ft, voltage drop is negligible (62A × ~1mΩ ≈ 0.06V); sizing is purely current-capacity driven. Trunk (Victron bar → ANL fuse → P115 split): 4 AWG at 62A. Per-element legs (P115 → DERNORD): 8 AWG at 31A. Fine-stranded welding cable or tinned marine copper; crimped lugs; no set-screw terminals in the power path.

- **Plumbing topology / Suburban:** confirmed the Suburban tank stays plumbed in. The triclamp is an additional under-counter stage, not a replacement. This bearing on the PRV question — the Suburban's existing PRV likely already covers the triclamp if no isolation valve separates them.

- **Three-state button / off mode:** off mode means relay de-energized, element off, tank cools to ambient. No target temperature is maintained in off mode. Two active setpoints: maintain ~80°F and boost ~104°F.

**Added as open tasks:**

- **SS304 vs SS316L:** vessel body is SS304; element sheath unconfirmed. 316L preferred for potable water contact. See Entry 20 for background.
- **Winterization / blowout:** element enters from the right end cap and sits at the lowest point of the tank. Main body should blow clear, but water pooled at the element base may not evacuate. Needs a confirmed procedure or drain port.
- **Stripboard vs breadboard:** build sequencing decision, not a design question. Park until tank and power circuit are assembled.
- **ESP32 firmware:** parked until hardware is assembled and tested.

---

## Entry 20 — SS304 vs SS316L for Potable Water Contact (resolved)

The triclamp vessel BOM currently specifies SS304 for the vessel body, and the DERNORD element sheath material is unconfirmed (likely SUS304). For drinking/cooking water contact, the relevant distinction:

- **SS304 (18-8):** standard food-grade stainless; widely used in brewing and food processing; fine for water contact in most conditions
- **SS316L (18-10-2 with molybdenum):** higher corrosion resistance, especially against chlorides. Preferred in marine and drinking water applications. The "L" (low-carbon) variant reduces sensitization risk from welding.

**Findings (2026-04-12):**

**DERNORD element sheath:** The product page describes the element as "stainless steel surface, electropolished" — likely generic marketing copy. Industry standard for DC/solar immersion heaters is Incoloy 800/825 sheath. Not worth confirming: given that water is used for hands and dishes only (no ingestion), sheath material is immaterial to safety.

**316L spool availability:** SS316L triclamp spools are available from multiple suppliers at modest premium over SS304:
- **[Sanitary Fittings](https://sanitaryfittings.us/product/tri-clamp-sanitary-spool):** both 304 and 316L; 2" 316L spool ~$160–165 for 12" (vs ~$150–155 for 304)
- **[GVC Direct](https://www.gvc.net/p/4203/316l-ss-sanitary-tri-clamp-spool-pieces):** 316L SS triclamp spools, 2" available, ~$135.80 base + $2.38/ft
- **[Glacier Tanks](https://www.glaciertanks.com/fittings/stainless-steel-316-fittings/tri-clamp-fittings/spools.html):** SS316 triclamp spools available in 2"
- DERNORD's own spool (dernord.com) is SS304 only

**TC × 1" FNPT adapter (element port):** DERNORD's adapter is SS304 only. Upgrading this to 316L requires sourcing from Sanitary Fittings or GVC Direct. This fitting has water contact on the inside (element insertion point) so it matters.

**Conclusion — SS304 DERNORD parts are acceptable; 316L is not a safety requirement:**

On further review, the concern is overstated for this application:

- SS304 is food-grade stainless, standard in brew kettles, dairy equipment, kitchen sinks, and restaurant equipment that contact potable water daily. No health risk at these temperatures.
- The Delta T warning ("304 corrodes in water") specifically applies to **heating element sheaths** at high watt density (high surface temperature) — not to vessel bodies sitting at 80–104°F.
- SS304 corrosion in low-chloride fresh municipal water at these temperatures occurs over years and manifests as pitting affecting longevity, not as harmful metal leaching into drinking water.
- RV use is intermittent, not continuous; feed water is typical municipal supply, not marine or brackish.
- This water is used for handwashing and dishes only — not consumed directly. No ingestion pathway; skin contact with any trace dissolved metals at realistic SS304 leaching rates is a non-issue.
- 316L parts are available from Sanitary Fittings and GVC Direct at modest premium (~$10–20 per fitting) if longevity is a concern in the future, but this is not a procurement blocker.

---

## Entry 21 — Relay Module Selection: Active-High vs Active-Low, Boot Safety (2026-04-12)

**Question:** The plan flagged the SRD-12VDC-SL-C relay module as TBD pending confirmation of active-high vs active-low behavior and safe boot state. If the module is active-low and the ESP32 GPIO floats low at power-on, the P115 contactor coil is briefly energized before firmware runs — element on before any temperature check.

**ESP32 board on hand:** AITRIP 30-pin ESP32-WROOM-32 DevKit, CP2102 USB-C. Standard WROOM-32 silicon; 30-pin layout exposes GPIO25 and GPIO26 among others.

**SRD-based boards rejected:** Cheap Chinese SRD-12VDC-SL-C optocoupler boards are typically active-low. Several ESP32 GPIOs pulse low briefly during boot and reset, which would briefly fire the relay. The firmware fix (`digitalWrite(pin, HIGH)` before `pinMode`) runs after the HAL initializes — not early enough to prevent the transient. Too fragile for a heating element circuit.

**Selected module: Pololu 2482** (Basic SPDT Relay Carrier with 12VDC Relay, assembled). Key properties:
- **Active-high:** relay coil fires when EN > 2.5V (up to 20V), driven by BSS138K N-channel MOSFET. Relay stays off when EN is low or floating.
- Most ESP32 GPIOs default low at boot → relay off before firmware runs. Safe by default.
- Coil powered separately via VDD/GND (12V); EN signal is the only GPIO connection.
- Omron G5LE-14-DC12 relay inside, 10A rated — more than adequate for the 12V P115 coil signal path.

**GPIO selection — 30-pin WROOM-32:**
- Use **GPIO25** and **GPIO26** (one per Pololu 2482, one per P115 contactor).
- Both confirmed present on the 30-pin layout; no boot-mode complications; not strapping pins.
- Avoid: GPIO0, GPIO2, GPIO5, GPIO12, GPIO15 (strapping pins); GPIO6–11 (flash); GPIO34/35/36/39 (input-only).

**Additional hardware mitigation:** Add a 10kΩ pull-down resistor from each EN pin to GND on the stripboard. This holds EN low during the brief window at power-on when the GPIO is in high-impedance (input) mode before firmware asserts it as OUTPUT LOW. Belt-and-suspenders against floating gate on the BSS138K (Pololu schematic not confirmed to include an internal pull-down).

**Firmware note:** In setup(), initialize both relay GPIOs LOW before calling pinMode(pin, OUTPUT). This ensures the pin state is set before the output driver is enabled.

**Conclusion:** Pololu 2482 × 2 selected, active-high, GPIO25 + GPIO26, 10kΩ pull-downs on stripboard. Relay module open question closed.

**BOM unchanged — SS304 DERNORD parts already on hand are fine.**

**Note:** The snap disc and NTC thermistor are surface-mounted on external copper nipples — no direct water contact with those sensors.

---

*Journal continues as design progresses. See `plan.md` for current state.*
