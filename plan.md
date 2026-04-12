# DC Water Heater — Current Plan

**Repo:** DC-water-heater  
**RV battery system:** 48V, 192Ah, 200A continuous delivery  
**Last updated:** see git log  

For full design history and decision rationale, see `explore.md`.

---

## System Architecture

Point-of-use heating under the counter at the kitchen faucet:

```
48V battery
    │
    ├─── [ANL fuse 80A] ─── [EV200 contactor] ─── [P115 contactor(s)] ─── DERNORD element(s)
    │                                                                                │
    │                                                                    Triclamp tank (pre-tempered store)
    │                                                                                │
    │                                                                         Faucet outlet
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
| Temp ports | ½" NPT thermowell in spool side ports |
| Clamps | 4× 2" hex bolt TC clamps |
| Gaskets | 4× EPDM (compliant, RV vibration tolerant) |
| Insulation | 2" foam pipe sleeve + end foam |

**Build task (not a design question):** complete triclamp tank detailed physical assembly.

---

## 2. Heating Element — SETTLED

**DERNORD 48V/1500W**, 1" NPSM thread, U-bend, Incoloy sheath, SS316 preferred (specify on order).

- Cold resistance: ~1.54Ω → 1500W at 48V, ~31A per element
- Two elements in parallel: ~0.77Ω → ~3000W at 48V
- Incoloy sheath electrically isolates energized conductor from water — eliminates DC electrolysis concern (see explore.md Entries 5 and 9)

**Build task:** measure actual cold resistance on delivery; verify sheath is SS316L not SUS304.

### Power Staging (with 2× DERNORD elements)

| Elements on | Power | ΔT at 0.5 GPM |
|---|---|---|
| 1 | 1,500W | +41°F |
| 2 | 3,000W | +82°F |

TMV (thermostatic mixing valve) downstream handles fine temperature control; contactors handle power steps. TMV essential when running both elements.

---

## 3. Control & Sensing — ACTIVE DESIGN WORK

### Architecture

```
48V bus → [ANL fuse 80A] → [EV200 contactor] → [P115 contactor(s)] → DERNORD element(s)
48V bus → [48V→12V buck] → ESP32 + relay module
Flow switch in series with EV200 coil → hard disconnect on flow stop, independent of ESP32
NTC thermistor (10kΩ @ 25°C, B=3950, M4 probe) at tank outlet
Snap disc thermostat (NC, opens at 110°F) in series with contactor coil signal — hardware safety cutoff
ESP32 thermostat: setpoint A ~80°F (maintain), setpoint B ~104°F (boost), one-button toggle
```

ESP32 acts as a configurable thermostat — reads outlet thermistor, energizes P115 contactor(s) via relay module to switch element power on/off. No PWM or FET in the power path. Staged power (1× or 2× elements via P115 contactors) selects heat rate; thermostat logic controls duty cycle within the active stage.

### Components

| Component | Part | Status |
|---|---|---|
| Microcontroller | ESP32 dev board | on hand |
| Master contactor | EV200 (12V coil, 500A rated) | on hand |
| Per-element contactor | P115BDA (12V coil, 50A rated) | on hand |
| Buck converter | Pololu D24V10F5 or equiv (48V→5V) | TBD |
| Thermistor | NTC 10kΩ @ 25°C, B=3950, M4 probe | TBD |
| Flow switch | TBD | TBD |
| Relay module | SRD-05VDC-SL-C (5V) | TBD |
| Snap disc thermostat | NC, opens at 110°F | TBD |

### Open Questions

- [ ] **Outlet thermistor placement:** where to locate on triclamp tank for accurate outlet temp reading
- [ ] **Minimum flow threshold:** flow switch must activate above dribble flow to prevent dry-fire; identify suitable paddle/flow switch and its threshold
- [ ] **Stripboard layout:** ESP32 driver circuit not yet produced
- [ ] **Hysteresis band:** characterize thermal lag at outlet thermistor vs element on/off at 0.5 GPM; set deadband wide enough to prevent rapid contactor cycling

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
