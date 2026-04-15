# DC Water Heater — Current Plan

**Repo:** DC-water-heater  
**RV battery system:** 48V nominal, 192Ah, 200A continuous delivery; 16S LiFePO4, 3.55V/cell max → **56.8V full-charge max**. All components in the power path must be rated for the full voltage range (nominal 48V, max 56.8V). Many DC elements and regulators are marketed at "48V" but rated only to 50–55V — always verify spec against 56.8V and prefer 60V-rated parts where available.  
**Last updated:** see git log  

For full design history and decision rationale, see `explore.md`.

---

## Problem Statement

The existing RV water heater is a Suburban propane/electric combo tank with several problems:

- **Standby heat loss:** the existing tank loses heat faster than expected — likely poor insulation; the propane flue running through the center may contribute but this is unconfirmed
- **Tank waste:** heating a full tank for small draws wastes energy
- **Cold purge waste:** hot water sitting in the pipe between tank and faucet is lost before hot water arrives
- **Maintenance burden:** anode rod management, seasonal draining for freeze protection

**Actual use pattern — short draws only:**

| Activity | Hot water needed | Notes |
|---|---|---|
| Hand wash | ~0.5 gal | |
| Dish wash (RV conservation pattern) | ~1.33 gal | 20 dishes × intermittent cycle (see below) |
| Short shower (5 min, low-flow) | ~5 gal | |

**RV dish-wash cycle (water-scarce pattern):** 4s wet draw → 5s washing → 4s rinse draw → 15s drying = 28s/dish. Flow is only 8s per dish (0.067 gal); the 20s off-time between draws is the critical feature — it allows the element to recover the tank to setpoint before the next draw. Total water for 20 dishes: ~1.33 gal (73% less than a continuous 5 gal pour). This is not a continuous draw and must not be modeled as one.

A hand wash draws 10× less water than the RV dish pattern; a short shower draws 4× more. These are all small, intermittent draws — not continuous high-flow demand.

**Why this rules out tankless:** at RV faucet flow rates (0.5–1.5 GPM), tankless requires 5–14 kW. At 48V DC that means 100–300A — not practical. Tanks tolerate slow recharge (1500W at 48V = ~31A); a pre-tempered tank delivers hot water instantly on demand from stored thermal energy. The RV use pattern — short draws spaced out over time — is exactly the pattern that makes tank-as-buffer work and tankless infeasible (see [electric heating rules of thumb](explore.md#electric-heating-rules-of-thumb) and [tube volume calculations](explore.md#tube-volume-flow-calculations) in explore.md).

**Goal:** Replace propane/electric system with a 48V DC under-counter tank, charged slowly from the existing battery, eliminating standby losses and cold purge delay.

---

## System Architecture

Point-of-use heating under the counter at the kitchen faucet:

```
48V battery
    │
    └─── [ANL fuse 80A] ─── [P115 contactor] ─── DERNORD element
                                                       │
                                             Triclamp tank (pre-tempered store)
                                                       │
                                                Faucet outlet

[Victron 12V rail] ─── ESP32 + relay module + sensors
```

**Under-counter triclamp tank:** pre-tempered water store, heated by DERNORD element, maintained at target temperature. Eliminates cold water purge at faucet.

---

## 1. Tank — SETTLED

Custom triclamp stainless vessel (see [triclamp vessel](explore.md#triclamp-vessel) in explore.md for rationale and full BOM):

| Component | Spec |
|---|---|
| Vessel body | 2" SS304 triclamp spool tube, ~10" |
| Element port (right end) | DERNORD 2" TC × 1" FNPT adapter |
| Left end cap | 2" TC blank ferrule |
| Inlet (bottom-left) | 2" TC spool with ½" NPT side port, PEX adapter |
| Outlet (top-right) | 2" TC spool with ½" NPT side port |
| Outlet copper nipple | ½" NPT × 2" copper nipple — thermal bridge for NTC thermistor and snap disc (copper conducts ~25× better than SS); sensors mount on nipple body under foam insulation |
| Faucet shutoff valve | Quarter-turn ball valve, ½" NPT × ½" NPT — after outlet nipple; allows heater isolation for maintenance without cutting all water | TBD |
| Faucet connection | ½" FIP → ⅜" OD compression braided supply line, shortest available run; insulate braided run (insulation type TBD — braided SS exterior differs from PEX) | TBD |
| Clamps | 4× 2" hex bolt TC clamps |
| Gaskets | 4× EPDM (compliant, RV vibration tolerant) |
| Insulation | 2" foam pipe sleeve + end foam |

**Build tasks (not design questions):**
- Complete triclamp tank detailed physical assembly
- Fit 2" copper nipple into outlet side port; mount bead NTC thermistor and snap disc on nipple body, both as close to tank wall as possible, with thermal paste + self-fusing silicone tape, then cover with foam insulation sleeve — sensors go under insulation so air-cooling is eliminated
- Install quarter-turn ball valve after outlet nipple; connect braided supply line (shortest available) from valve to faucet hot-water port; insulate braided run (insulation type TBD)
- Insulate supply pipes between new under-counter triclamp tank and the old Suburban tank — uninsulated pipes are a standby heat loss path
  - **Purchase:** PATIKIL 2" ID (51mm) foam pipe insulation (Lowe's) for the 2" OD triclamp spool body — standard IPS-sized "2 inch pipe" insulation is too large (fits 2.375" OD, not 2.00")
  - **Purchase:** Frost King or Everbilt ½" foam pipe insulation (Home Depot or Lowe's) for ½" PEX supply runs
  - **Future task:** determine appropriate insulation for braided stainless supply line (braided SS exterior not compatible with standard foam sleeve adhesion)
  - **Future task:** hands-free faucet — keep existing faucet; add NC solenoid valve (½" NPT, 12V DC, potable-water rated) between tank outlet ball valve and braided supply line; add LD2410 mmWave sensor co-located with the 3-mode switch at cabinet front; ESP32 relay channel opens solenoid while hand is detected + N-second hold timer. Pre-set faucet handle to full hot. Powered from 12V Victron rail. Foot pedal ruled out (see [foot pedal analysis](explore.md#hands-free-foot-pedal-ruled-out) in explore.md); commercial touchless faucet (Moen MotionSense Wave) remains fallback if DIY path proves problematic. TBD: solenoid part number, sensor mount position and range gate settings (verify at commissioning), firmware implementation (park until hardware assembled). See [touchless options](explore.md#hands-free-touchless-options) and [DIY solenoid path](explore.md#hands-free-diy-solenoid-mmwave) in explore.md.

---

## 2. Heating Element — SETTLED

**DERNORD 48V/1500W**, 1" NPSM thread, U-bend.

- Cold resistance: ~1.54Ω → 1500W at 48V, ~31A
- Incoloy sheath electrically isolates energized conductor from water — eliminates DC electrolysis concern (see [DC electrolysis hard stop](explore.md#dc-electrolysis-rewind-hard-stop) and [DC safety confirmation](explore.md#dc-power-safety-research) in explore.md)

**Build task:** measure actual cold resistance on delivery.

This is ordered, but currently have a 1000W dernord element with same thread.

**Build sequencing:** Single-element system first. A second element would require a second triclamp tank (the existing tank has only one element port); that is a significant plumbing addition. See Second Element — Parked section.

---

## 3. Cabling — SETTLED

**Supply run:** Victron distribution bar → ANL fuse → P115 contactor → DERNORD element. Distance from Victron bar to the heating circuit is ~2 ft.

At 2 ft, voltage drop is negligible regardless of gauge (31A × ~1 mΩ round-trip ≈ 0.03V). Wire sizing is current-capacity driven, not drop-driven.

| Segment | Max current | Recommended gauge | Notes |
|---|---|---|---|
| Victron bar → ANL fuse | 31A | 4 AWG fine-stranded | Marine/welding grade; ≤18" from battery terminal to fuse; 80A ANL fuse sized conservatively (handles second-element path if ever added) |
| ANL fuse → P115 contactor | 31A | 4 AWG | |
| P115 → DERNORD element | 31A | 8 AWG | |

Fine-stranded welding cable or marine-grade tinned copper preferred for RV flex/vibration tolerance. Lugs crimped and heat-shrunk; no set-screw terminals in the power path.

**Control wiring** (12V coil signal, ESP32 GPIO) is low-current and can run 22–18 AWG.

---

## Second Element — PARKED

Adding a second DERNORD 48V/1500W element would require a **second triclamp tank** joined to the first — the existing tank has only one element port and cannot accommodate a second U-bend. This is a significant plumbing addition; revisit only if the single-element system proves demonstrably insufficient for actual use patterns.

### Power Staging reference (two-tank / two-element scenario)

| Elements on | Power | ΔT at 0.25 GPM | ΔT at 0.5 GPM |
|---|---|---|---|
| 1 | 1,500W | +41°F | +20.5°F |
| 2 | 3,000W | +82°F | +41°F |

If a second element is ever added:
- Second P115BDA contactor (purchase required)
- Second Pololu 2482 relay module (GPIO26, 10kΩ pull-down to GND)
- Second triclamp tank with element port; join to first tank via ½" PEX
- Cabling: run second 8 AWG leg from second P115; 4 AWG trunk to ANL fuse handles 62A combined
- Safety chain: put snap disc and pressure switch on the **common 12V supply rail upstream of both relay modules** — so either safety device simultaneously cuts both elements

---

## 4. Control & Sensing — ACTIVE DESIGN WORK

### Architecture

```
48V bus → [ANL fuse 80A] → [P115 contactor] → DERNORD element

[Victron 12V rail] → ESP32 + relay module
Coil signal chain: ESP32 GPIO (active-high) → Pololu 2482 (COM→NO contacts) → snap disc NC → pressure switch NO → P115 coil → GND
Snap disc (NC, opens at 113°F / 45°C) — hardware thermal safety cutoff, in series with coil signal
Pressure switch (NO, closes ≥10–15 psi) — dry-fire interlock, in series with coil signal
NTC thermistor (10kΩ @ 25°C, B=3950, bead type) on outlet copper nipple → ESP32 ADC
  NTC wiring: 3.3V → [fixed resistor] → ADC pin → NTC → GND (voltage divider; resistor value TBD — see open questions)
ESP32 thermostat: setpoint A ~80°F (maintain), setpoint B ~104°F (boost), three-position switch (off / maintain / boost)
Ground: Victron 12V rail shares common negative bus with 48V battery — P115 coil, relay module, and ESP32 all reference the same negative.
```

One P115 contactor for the single DERNORD element. ESP32 reads outlet thermistor and signals the contactor coil via relay module — no FET or PWM in the power path.

### Components

| Component | Part | Status |
|---|---|---|
| Microcontroller | AITRIP ESP32-WROOM-32, 30-pin DevKit, CP2102 USB-C (ESP-WROOM-32) | on hand |
| Contactor | P115BDA (12V coil, 50A rated) | on hand |
| Control supply | Existing Victron 12V rail (360W, lightly loaded) — **settled**. Fallback if self-contained 48V supply ever needed: Pololu D45V5F12 (57V max in, 12V out) or APM81815 (#5269, 12.1–72V in, 12V/0.8A out); see [control supply options](explore.md#control-supply-12v-rail) in explore.md. | Settled |
| Thermistor | AILEWEI MF52A, NTC 10kΩ @ 25°C, B=3950, ~2.5mm bead, tinned copper leads ≥25mm; see [NTC thermistor selection](explore.md#ntc-thermistor-mf52a) in explore.md. Wired as voltage divider with a fixed resistor to 3.3V — see open questions for resistor value. | Ordered |
| Relay module (contactor) | Pololu 2482 (Basic SPDT Relay Carrier, 12VDC, active-high EN pin) × 1; use **COM→NO contacts** — element is off when EN=LOW (boot-safe by design). See [relay module boot safety](explore.md#relay-module-boot-safety) in explore.md. | TBD |
| Pull-down resistors | 10kΩ × 1 (one per Pololu 2482 EN pin, mounted on stripboard between EN and GND) — belt-and-suspenders against floating gate at power-on; see [relay module boot safety](explore.md#relay-module-boot-safety) in explore.md | TBD |
| Snap disc thermostat | NC, opens at 113°F (45°C) — see [sensor mounting and snap disc trip](explore.md#sensor-format-snap-disc-trip) in explore.md for trip point rationale and TMV contingency | TBD |
| Pressure switch | NO, ¼" NPT, set point 10–15 psi, potable-water rated | TBD |
| Inlet tee + bushing | ½" NPT street tee (one male end to screw into tank inlet port, two female) + ½"→¼" NPT bushing for pressure switch branch; run port to PEX supply adapter | TBD |
| 3-mode switch | Physical switch for off / maintain / boost states; panel-mount at cabinet front; wiring to ESP32 GPIO. Type and part TBD — needs to latch in three positions (e.g., 3-position rotary or rocker; momentary pushbutton requires firmware state machine). | TBD |
| Faucet water valve (hands-free faucet — parked) | NC solenoid water valve, ½" NPT, 12V DC, potable-water rated; mounted at tank outlet after ball valve. Driven by a **separate, dedicated ESP32 relay channel** (requires an additional Pololu 2482 relay module beyond the ×1 contactor relay above). This is a water valve, not a relay — distinct from the contactor relay module. Part TBD. | TBD |
| mmWave sensor (hands-free faucet) | LD2410, ~22 × 16 mm, UART to ESP32; co-mounted with 3-mode switch at cabinet front. ~$5–8. Range gate settings TBD at commissioning. | TBD |

### Open Questions

- [ ] **3-mode switch spec:** choose physical switch type for off / maintain / boost. A 3-position latching rotary or rocker switch is simplest (each position directly maps to a GPIO state, no firmware debounce logic needed). A momentary pushbutton cycles through states in firmware but requires more code. Decide part and panel-mount approach; needs to fit cabinet-front panel alongside LD2410 sensor.
- [ ] **Stripboard layout:** decide whether to build ESP32 driver circuit on stripboard now or breadboard for initial testing; not a design question, a build sequencing decision
- [ ] **Hysteresis band:** characterize thermal lag at outlet thermistor vs element on/off at 0.5 GPM; set deadband wide enough to prevent rapid contactor cycling
- [x] **Pressure switch / water pump interaction:** Pump off → pressure bleeds → heater shuts off. Acceptable behavior — if the pump is off, no water is being drawn and hot water maintenance is unnecessary. The pressure switch doubles as a "system awake" interlock. Do not have ESP32 drive the pump on when pressure drops. See [pump interaction analysis](explore.md#pressure-switch-pump-interaction) in explore.md.
- [x] **PRV (pressure relief valve):** No dedicated PRV needed. Open system (no check valve) means thermal expansion pushes back upstream in normal operation. Suburban's existing T&P valve covers the full interconnected pressure zone. Constraint: never add an isolation valve between the Suburban and triclamp — that would create a separate zone requiring its own PRV. See [PRV not needed](explore.md#prv-not-needed) in explore.md.
- [x] **Winterization / blowout:** the triclamp tank sits 2' above the Suburban; most water drains back by gravity on winterization. Residual ~1" at the element base is not a freeze risk — air headspace in the upper half of the spool absorbs expansion, Incoloy sheath is not damaged by external ice, and no sealed cavity forms. No drain port or special procedure needed. Removing the element (same as Suburban practice) fully drains if desired. See [winterization](explore.md#winterization) in explore.md.
- [ ] **NTC voltage divider fixed resistor:** NTC is wired as a voltage divider (3.3V → fixed R → ADC → NTC → GND). The standard choice is 10kΩ (matches NTC nominal resistance, puts midpoint at 1.65V at 25°C, good ADC sensitivity across 80–113°F range). Confirm this value, add it to the stripboard BOM, and document it before firmware ADC conversion math is written — the Steinhart-Hart R calculation depends on knowing the divider resistance. First use of a voltage divider with an NTC; explore.md can record findings once tested.
- [ ] **ESP32 VIN at 12V:** confirm AITRIP 30-pin DevKit accepts 12V on VIN without damage before first power-on; check board schematic or spec sheet against the onboard regulator's maximum input rating.
- [ ] **LD2410 UART pin assignment (future):** ESP32 UART1 default pins GPIO9/10 conflict with flash memory lines. When implementing LD2410, remap UART to GPIO16/17 (UART2) or other available pins. Plan GPIO assignments for relay GPIOs (25, 26), 3-mode switch, and LD2410 UART as a complete group before firmware work begins.
- [ ] **ESP32 firmware (future task, park until hardware assembled):** thermostat logic, NTC Steinhart-Hart ADC conversion, hysteresis band, three-position switch (off / maintain ~80°F / boost ~104°F), relay drive, watchdog timer. Off mode: relay de-energized, element off, tank cools to ambient — no target temperature maintained. **Important boot sequence:** in setup(), set both relay GPIO pins LOW before calling pinMode(pin, OUTPUT) to ensure element is off before the output driver enables.
- [x] **Model boost warm-up and recovery time:** Modeling assumptions: inlet 50°F, flow 0.5 GPM, tank 0.119 gal (0.99 lb water + 1.12 lb SS tube = 1.13 BTU/°F thermal mass). Tank time constant during flow: τ = 16 sec.

  **Static warm-up (no flow), maintain 80°F → boost 104°F:**
  | Elements | Time |
  |---|---|
  | 1× | ~19 sec |
  | 2× | ~10 sec |

  **Outlet temp during continuous draw (flow-through steady state):**
  | Elements | Steady-state outlet | Time to reach steady state |
  |---|---|---|
  | 1× | ~71°F | ~1 min |
  | 2× | ~91°F | ~1 min |

  **RV dish-wash cycle analysis (1× element, boost setpoint 104°F):** Per-dish cycle is self-resetting — tank recovers to setpoint within the 15s drying phase (only 7s of heating needed), leaving 8s of margin before the next dish. Outlet temps throughout a 20-dish session: **96–104°F on every draw**. Single element is sufficient for this pattern.

  **Conclusion:** single-element performance is sufficient for the actual RV use pattern. The continuous-draw model (5 gal uninterrupted) is not representative. Build and commission single-element system first as planned. Power staging table moved to Second Element — Parked section.

---

## Key Decisions and Rationale (summary)

| Decision | Rationale |
|---|---|
| Resistive not induction | Simpler electronics; Incoloy sheath provides adequate isolation; see [induction heating deep dive](explore.md#induction-heating) in explore.md |
| Sheathed not bare wire | DC electrolysis drives toxic ion dissolution from bare nichrome; sheath eliminates concern; see [DC electrolysis hard stop](explore.md#dc-electrolysis-rewind-hard-stop) and [DC safety confirmation](explore.md#dc-power-safety-research) in explore.md |
| 48V direct not 96V boost | Thermal storage (tank) is cheaper and simpler than boost converter; see [Elex/Rheem teardown](explore.md#elex-teardown-rheem-vessel) in explore.md |
| Triclamp SS not copper pipe | Lower standby heat loss; no soldering; fully disassemblable; see [triclamp vessel](explore.md#triclamp-vessel) in explore.md |
| Triclamp tank not RTEX vessel | RTEX copper tubes ~1¼" diameter — too little water volume; see [Elex/Rheem teardown](explore.md#elex-teardown-rheem-vessel) in explore.md |
| Triclamp tank not Fogatti drop-in | Custom triclamp vessel better fits under-counter two-stage architecture; see [faucet heater full design](explore.md#incoloy-faucet-heater-full-design) in explore.md |
| No propane water heating | Propane generator (if needed) charges 48V battery; single DC path serves all heating |
| Contactors not FET/PWM for power switching | Resistive load; on/off thermostat via contactor is simpler, no gate drive problem, avoids PWM complexity; see [control simplification](explore.md#control-simplification-thermostat) in explore.md |
| P115 not EV200; no flow switch | Tank design — element always submerged, flow events not a dry-fire risk; EV200 justified only by flow switch role, now removed; P115 right-sized at 50A for 31A element load; see [flow switch removed](explore.md#flow-switch-removed) in explore.md |
| Pressure switch not thermal fuse for dry-fire protection | Thermal fuse (Option A) requires a new tank port; pressure switch on inlet line (Option B) requires no tank modification; NO switch fails safe (hot water loss, not element damage); see [dry-fire analysis](explore.md#dry-fire-snap-disc-inadequate) and [pressure switch solution](explore.md#dry-fire-pressure-switch) in explore.md |
