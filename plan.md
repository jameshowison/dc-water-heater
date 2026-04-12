# DC Water Heater — Current Plan

**Repo:** DC-water-heater  
**RV battery system:** 48V, 192Ah, 200A continuous delivery  
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

| Activity | Hot water needed |
|---|---|
| Hand wash | ~0.5 gal |
| Dish basin | ~5 gal |
| Short shower (5 min, low-flow) | ~5 gal |

A hand wash draws 10× less water than a dish basin; a short shower and a dish basin are the same volume. These are all small, intermittent draws — not continuous high-flow demand.

**Why this rules out tankless:** at RV faucet flow rates (0.5–1.5 GPM), tankless requires 5–14 kW. At 48V DC that means 100–300A — not practical. Tanks tolerate slow recharge (1500W at 48V = ~31A); a pre-tempered tank delivers hot water instantly on demand from stored thermal energy. The RV use pattern — short draws spaced out over time — is exactly the pattern that makes tank-as-buffer work and tankless infeasible (see explore.md Entries 3 and 11).

**Goal:** Replace propane/electric system with a 48V DC under-counter tank, charged slowly from the existing battery, eliminating standby losses and cold purge delay.

---

## System Architecture

Point-of-use heating under the counter at the kitchen faucet:

```
48V battery
    │
    ├─── [ANL fuse 80A] ─── [P115 contactor] ─── DERNORD element
    │                  └─── [P115 contactor] ─── DERNORD element   (optional second element)
    │                                                   │
    │                                         Triclamp tank (pre-tempered store)
    │                                                   │
    │                                            Faucet outlet
    │
    └─── [48V→12V buck] ─── ESP32 + relay module + sensors
```

**Under-counter triclamp tank:** pre-tempered water store, heated by DERNORD element, maintained at target temperature. Eliminates cold water purge at faucet.

---

## 1. Tank — SETTLED

Custom triclamp stainless vessel (see explore.md Entry 8 for rationale and full BOM):

| Component | Spec |
|---|---|
| Vessel body | 2" SS304 triclamp spool tube, ~14" |
| Element port (right end) | DERNORD 2" TC × 1" FNPT adapter |
| Left end cap | 2" TC blank ferrule |
| Inlet (bottom-left) | 2" TC spool with ½" NPT side port, PEX adapter |
| Outlet (top-right) | 2" TC spool with ½" NPT side port, 3/8" faucet fitting |
| Sensor nipples | ½" NPT × ~1" copper nipple on inlet and outlet side ports — surface-mount point for NTC thermistor and snap disc |
| Clamps | 4× 2" hex bolt TC clamps |
| Gaskets | 4× EPDM (compliant, RV vibration tolerant) |
| Insulation | 2" foam pipe sleeve + end foam |

**Build tasks (not design questions):**
- Complete triclamp tank detailed physical assembly
- Fit copper nipples in inlet and outlet side ports; surface-mount NTC thermistor on outlet nipple and snap disc on outlet nipple, both with thermal paste + self-fusing silicone tape
- Insulate supply pipes between new under-counter triclamp tank and the old Suburban tank — uninsulated pipes are a standby heat loss path

---

## 2. Heating Element — SETTLED

**DERNORD 48V/1500W**, 1" NPSM thread, U-bend.

- Cold resistance: ~1.54Ω → 1500W at 48V, ~31A per element
- Two elements in parallel: ~0.77Ω → ~3000W at 48V
- Incoloy sheath electrically isolates energized conductor from water — eliminates DC electrolysis concern (see explore.md Entries 5 and 9)

**Build task:** measure actual cold resistance on delivery.

This is ordered, but currently have a 1000W dernord element with same thread.

### Power Staging (with 2× DERNORD elements)

| Elements on | Power | ΔT at 0.5 GPM |
|---|---|---|
| 1 | 1,500W | +41°F |
| 2 | 3,000W | +82°F |


---

## 3. Cabling — SETTLED

**Supply run:** Victron distribution bar → ANL fuse → P115 contactor(s) → DERNORD element(s). Distance from Victron bar to the heating circuit is ~2 ft.

At 2 ft, voltage drop is negligible regardless of gauge (62A × ~1 mΩ round-trip ≈ 0.06V). Wire sizing is current-capacity driven, not drop-driven.

| Segment | Max current | Recommended gauge | Notes |
|---|---|---|---|
| Victron bar → ANL fuse | 62A (both elements) | 4 AWG fine-stranded | Marine/welding grade; ≤18" from battery terminal to fuse |
| ANL fuse → P115 contactors (trunk) | 62A | 4 AWG | Trunk before split |
| P115 → each DERNORD element | 31A | 8 AWG | One run per element/contactor |

Fine-stranded welding cable or marine-grade tinned copper preferred for RV flex/vibration tolerance. Lugs crimped and heat-shrunk; no set-screw terminals in the power path.

**Control wiring** (12V coil signal, ESP32 GPIO) is low-current and can run 22–18 AWG.

---

## 4. Control & Sensing — ACTIVE DESIGN WORK

### Architecture

```
48V bus → [ANL fuse 80A] → [P115 contactor] → DERNORD element
                       └→ [P115 contactor] → DERNORD element   (optional second element)
48V bus → [48V→12V buck] → ESP32 + relay module
ESP32 → relay module → 12V coil signal → P115 contactor(s)
Snap disc (NC, opens at 110°F) in series with 12V coil signal — hardware safety cutoff
NTC thermistor (10kΩ @ 25°C, B=3950, M4 probe) at tank outlet → ESP32 ADC
ESP32 thermostat: setpoint A ~80°F (maintain), setpoint B ~104°F (boost), three state-button (off, maintain, boost). 
```

One P115 contactor per element; 1 or 2 elements fitted depending on heating rate needed. ESP32 reads outlet thermistor and signals contactor coil(s) via relay module — no FET or PWM in the power path.

### Components

| Component | Part | Status |
|---|---|---|
| Microcontroller | AITRIP ESP32-WROOM-32, 30-pin DevKit, CP2102 USB-C (ESP-WROOM-32) | on hand |
| Contactor (one per element) | P115BDA (12V coil, 50A rated) | on hand |
| Buck converter | 48V→12V, ≥58V input rating (e.g. Pololu D57V45F12 or equiv) | TBD |
| Thermistor | NTC 10kΩ @ 25°C, B=3950, M4 probe | TBD |
| Relay module | Pololu 2482 (Basic SPDT Relay Carrier, 12VDC, active-high EN pin) × 2 — one per contactor | TBD |
| Snap disc thermostat | NC, opens at 110°F | TBD |

### Open Questions

- [ ] **Water level / dry-fire protection:** UNDECIDED. The snap disc on the outlet copper nipple is NOT adequate for dry-fire protection — it measures water temperature and has no thermal path to the element in a dry-fire condition (see explore.md Entry 22). The DERNORD 48V/1500W element has no built-in ECO disc. Best option is a thermal fuse in contact with the element sheath (industrial standard practice), but this requires a thermowell with a dedicated NPT port — current tank BOM has both side ports spoken for. **Next: explore Option B — pressure switch in the supply line wired in series with coil signal** as an alternative that avoids the port constraint. Also explore float switch options inside the 2" triclamp vessel. Also explore how home brewing/fermentation heating avoids dry-firing (similar small vessel + immersion element problem).
- [x] **Outlet thermistor placement:** resolved — NTC surface-mounted on copper outlet nipple with thermal paste and self-fusing silicone tape (see Sensor nipples in BOM above)
- [ ] **Stripboard layout:** decide whether to build ESP32 driver circuit on stripboard now or breadboard for initial testing; not a design question, a build sequencing decision
- [ ] **Hysteresis band:** characterize thermal lag at outlet thermistor vs element on/off at 0.5 GPM; set deadband wide enough to prevent rapid contactor cycling
- [x] **Relay module active-high vs active-low:** resolved — use Pololu 2482 (active-high, BSS138K MOSFET driver, EN threshold 2.5V). Relay stays off when EN is low or floating. On the 30-pin WROOM-32 board, most GPIOs boot low; drive from GPIO25 and GPIO26 (safe output pins, not strapping pins, confirmed present on 30-pin board). Add a 10kΩ pull-down from each EN pin to GND on the stripboard as belt-and-suspenders against any float at power-on. In firmware: initialize pin LOW before setting OUTPUT mode. Avoid strapping pins GPIO0, GPIO2, GPIO5, GPIO12, GPIO15. Also avoid GPIO6–11 (flash), GPIO34/35/36/39 (input-only).
- [ ] **PRV (pressure relief valve):** Entry 12 flags PRV as good practice for thermal expansion. The Suburban tank is staying plumbed in (triclamp is an additional under-counter stage, not a replacement). Explore: (a) whether the Suburban's existing PRV already covers the triclamp vessel — likely yes if both share the common supply line with no isolation valve between them; (b) if a dedicated PRV is needed, where it would drain (under-counter drain pan? existing Suburban drain line?); (c) whether the open RV system makes thermal expansion a non-issue in normal operation, leaving only the both-valves-closed edge case to design for.
- [x] **SS304 vs SS316L for potable water:** resolved — SS304 DERNORD parts are fine. 316L is a longevity improvement, not a safety matter. Water is used for hands and dishes only (no ingestion), SS304 is food-grade, and temperatures are low (80–104°F). See explore.md Entry 20.
- [ ] **Winterization / blowout:** the DERNORD element enters from the right end cap and is the lowest point in the tank. A compressed-air blowout from the inlet port should clear the main tank body, but water pooled at and around the element base may not fully evacuate. Confirm whether residual water at the element poses freeze damage risk, and whether a drain port or specific blowout procedure is needed (e.g. blow from outlet, drain from inlet, or accept a small residual volume).
- [ ] **ESP32 firmware (future task, park until hardware assembled):** thermostat logic, NTC Steinhart-Hart ADC conversion, hysteresis band, three-state button (off / maintain ~80°F / boost ~104°F), relay drive, watchdog timer. Off mode: relay de-energized, element off, tank cools to ambient — no target temperature maintained.

---

## Key Decisions and Rationale (summary)

| Decision | Rationale |
|---|---|
| Resistive not induction | Simpler electronics; Incoloy sheath provides adequate isolation; see explore.md Entry 10 |
| Sheathed not bare wire | DC electrolysis drives toxic ion dissolution from bare nichrome; sheath eliminates concern; see explore.md Entries 5 and 9 |
| 48V direct not 96V boost | Thermal storage (tank) is cheaper and simpler than boost converter; see explore.md Entry 4 |
| Triclamp SS not copper pipe | Lower standby heat loss; no soldering; fully disassemblable; see explore.md Entry 8 |
| Triclamp tank not RTEX vessel | RTEX copper tubes ~1¼" diameter — too little water volume; see explore.md Entry 4 |
| Triclamp tank not Fogatti drop-in | Custom triclamp vessel better fits under-counter two-stage architecture; see explore.md Entry 6 |
| No propane water heating | Propane generator (if needed) charges 48V battery; single DC path serves all heating |
| Contactors not FET/PWM for power switching | Resistive load; on/off thermostat via contactor is simpler, no gate drive problem, avoids PWM complexity; see explore.md Entry 15 |
| P115 not EV200; no flow switch | Tank design — element always submerged, flow events not a dry-fire risk; EV200 justified only by flow switch role, now removed; P115 right-sized at 50A for 31A element load; see explore.md Entry 17 |
