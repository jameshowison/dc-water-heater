# DC Water Heater — Current Plan

**Repo:** DC-water-heater  
**RV battery system:** 48V, 192Ah, 200A continuous delivery  
**Last updated:** see git log  

For full design history and decision rationale, see `explore.md`.

---

## System Architecture

Two-stage point-of-use heating under the counter at the kitchen faucet:

```
48V battery
    │
    ├─── [ANL fuse 80A] ─── [EV200 contactor] ─── [MOSFET] ─── DERNORD element(s)
    │                                                                    │
    │                                                             Triclamp tank (pre-tempered store)
    │                                                                    │
    │                                                         Faucet body heater (2" SS tube, ~840mL)
    │                                                                    │
    │                                                              Faucet outlet
    │
    └─── [48V→12V buck] ─── ESP32 + contactor coil + sensors
```

**Stage 1 — Under-counter triclamp tank:** pre-tempered water store, heated by DERNORD element, maintained at target temperature. Eliminates cold water purge at faucet.

**Stage 2 — Faucet body heater:** in-line boost at point of use, 2" SS sanitary tube body ~14" long (~840mL), DERNORD element, Armaflex insulated. Activates predictively via PIR; gate held closed until water is ready.

---

## Components

### Heating Elements
- **DERNORD 48V/1500W**, 1" NPSM thread, U-bend, Incoloy or SS316 sheath
- Cold resistance: ~1.54Ω → 1500W at 48V, ~31A per element
- Two elements in parallel: ~0.77Ω → ~3000W at 48V (verify measured resistance)
- Used in both the tank and the faucet body

### Triclamp Tank (under-counter)
- Custom vessel: stainless sanitary tube/spool components with triclamp fittings
- DERNORD element via triclamp port (exact configuration TBD — offline design in progress)
- Insulated externally; freeze protection via drain valve accessible from outside
- [photo: chat 990abbdd]

### Faucet Body Heater
- Body: 2" stainless sanitary tube, ~14" long
- Element: DERNORD 48V/1500W, terminals at bottom
- Insulation: Armaflex wrap
- Mounting: vertical, under counter
- Gate: NC solenoid, held open only when water is at temperature

### Contactors (on hand)
- **EV200** (Gigavac/Tyco): 12V coil, 48V/500A rated — master safety disconnect; flow switch wired in series with coil circuit so flow stop = hard disconnect regardless of ESP32 state
- **P115BDA** (12V coil) or **P115FDA** (48V coil): 50A rated — per-element switching; staged power control
- Both EV200 and P115 have built-in coil suppression — no external flyback diode needed

### Control Electronics
- **ESP32** dev board (on hand): thermostat logic, PID staging, safety interlocks
- **5V relay module** (SRD-05VDC-SL-C): low-current signal switching
- **MOSFET** (IRFP4568 or equiv, TO-247): element power switching — ⚠️ not yet on hand; gate drive needs verification (ESP32 GPIO is 3.3V, Vgs(th) ~4V — may need gate driver IC)
- **48V→12V buck** (Pololu D24V10F5 or equiv): powers ESP32 + contactor coils
- **NTC thermistors** (10kΩ @ 25°C, B=3950, M4 probe): inlet and outlet temperature sensing
- **Flow switch**: activates EV200, triggers ESP32 heating logic
- **PIR sensor**: predictive pre-heating before faucet use
- **Lever microswitch**: faucet handle detection

### Temperature Safety
- Snap disc thermostat (NC, opens at 110°F): hardware-level safety cutoff, wired in series with contactor coil signal
- Clamped to copper nipple with thermal paste + self-fusing silicone tape

---

## Power Staging (with 2× DERNORD elements)

| Elements on | Power | ΔT at 0.5 GPM |
|---|---|---|
| 1 | 1,500W | +41°F |
| 2 | 3,000W | +82°F |

TMV (thermostatic mixing valve) downstream handles fine temperature control; contactors handle power steps. TMV essential when running both elements.

---

## Open Items

- [ ] Triclamp tank detailed design (offline, to be documented here)
- [ ] MOSFET gate drive: confirm ESP32 3.3V GPIO fully enhances IRFP4568, or add gate driver IC
- [ ] Measure actual DERNORD element resistance (cold and at operating temp)
- [ ] Confirm DERNORD sheath material suitable for potable water (SS316L preferred over SUS304)
- [ ] Clarify electrical isolation of sheathed element from water — bonding/grounding requirements
- [ ] Electrolysis risk assessment at 48V DC with sheathed elements in potable water
- [ ] Outlet thermistor placement on triclamp tank
- [ ] Minimum flow threshold for flow switch to avoid dry-fire at dribble flow
- [ ] Stripboard layout for ESP32 driver circuit (not yet produced)
- [ ] PID tuning: thermal lag between element power change and outlet thermistor at 0.5 GPM

---

## Key Decisions and Rationale (summary)

| Decision | Rationale |
|---|---|
| Resistive not induction | Simpler electronics; Incoloy sheath provides adequate isolation; see explore.md Entry 2 |
| 48V direct not 96V boost | Thermal storage (tank) is cheaper and simpler than boost converter; see explore.md Entry 6 |
| Triclamp tank not RTEX vessel | RTEX copper tubes ~1¼" diameter — too little water volume; see explore.md Entry 6 |
| Triclamp tank not Fogatti drop-in | Custom triclamp vessel better fits under-counter two-stage architecture |
| No propane water heating | Propane generator (if needed) charges 48V battery; single DC path serves all heating |
| Staged contactors not PWM | Resistive load; binary staging + downstream TMV is simpler and avoids contactor cycle wear |
