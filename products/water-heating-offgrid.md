# Hot Water, Cold Math: Rules of Thumb for Heating Water Off-Grid

*Or: why your RV's 5-gallon tank might already be the answer*

---

## The Argument That Started This

It began, as many engineering rabbit holes do, with mild irritation.

Someone in an off-grid forum insisted — with the confidence of a person who has Never Been Wrong — that you absolutely need a full five gallons of hot water to wash dishes. In an RV. For two people. After a quiet dinner.

Now, five gallons is 18 liters. That's the kind of quantity you'd use to mop a floor, not rinse a couple of plates and a pan. But the claim kept coming up, and nobody seemed to have numbers to push back with. Just vibes. Hot, wasteful vibes.

Around the same time, I'd been experimenting with an impulse cooktop — one of those induction burners designed for off-grid use that leans on a battery buffer rather than drawing directly from shore power or a large inverter. The concept is elegant: charge a buffer battery slowly from your 52V system, then dump it fast when you need to boil water. The battery does the heavy lifting; your solar or alternator just has to keep up on average, not at peak.

And I started wondering: could you do the same thing for hot water?

That question turned into a surprisingly deep dive into the physics of heating water with electricity. What follows are the rules of thumb I developed along the way — some found, some derived from first principles — and the conclusions they led to. The math is real but the arithmetic is friendly. Bring a cup of tea.

---

## The Physics, in One Sentence

Resistance heating is 100% efficient. Every watt you put in becomes heat. There are no combustion losses, no flue gases, no pilot lights burning away while you sleep. This makes the math unusually clean: energy in equals energy stored in the water, full stop.

The fundamental relationship is just the specific heat of water:

> **It takes 1 BTU to raise 1 pound of water by 1°F.**

This is, delightfully, almost the definition of a BTU — so it's exact. In metric, it's 4,186 joules per kilogram per degree Celsius, which works out to **1.16 Wh per liter per °C**.

---

## How Much Hot Water Do You Actually Use?

Before we get to the electricity, let's anchor the volumes. These are hot water drawn (not total flow, which is mixed with cold):

| Activity | Gallons (hot) |
|---|---|
| Hand wash (hands) | 0.5 |
| Dish basin (efficient hand washing) | 5 |
| Dishwasher (modern) | 4 |
| Short shower (5 min, low-flow) | 5 |
| Long shower (10 min, standard) | 15 |

That dish basin number is the one that started this whole investigation. Five gallons is a lot — but it's also the upper bound for a thorough hand-washing session, not a minimum. A basin with a couple of inches of hot water handles most RV dish loads in well under a gallon.

---

## The Energy Crosstab

Energy to heat hot water, US customary, resistance electric (COP=1):

| Activity → | Hand wash | Dish basin | Dishwasher | Short shower | Long shower |
|---|---|---|---|---|---|
| **Gallons** | **0.5** | **5** | **4** | **5** | **15** |
| Summer (ΔT ≈ 50°F, ~70°F inlet) | 0.06 kWh | 0.6 kWh | 0.5 kWh | 0.6 kWh | 1.8 kWh |
| Average (ΔT ≈ 60°F) | 0.07 kWh | 0.7 kWh | 0.6 kWh | 0.7 kWh | 2.2 kWh |
| Winter (ΔT ≈ 80°F, ~40°F inlet) | 0.10 kWh | 1.0 kWh | 0.8 kWh | 1.0 kWh | 2.9 kWh |

Winter is about **1.5× summer** — not quite double, but close enough that "winter costs half again as much" is a useful rule.

**Round number takeaways:**
- A short shower or a dish basin ≈ half a kWh (summer) to 1 kWh (winter)
- A long shower ≈ 2–3 kWh depending on season
- Dishwasher machine wash ≈ dish basin by hand — that's the water efficiency story

---

## Tank vs. Tankless: Where the Real Pain Lives

A storage tank heats water slowly and holds it. A tankless heater heats water as it flows. The physics of each is completely different, and this is where the 52V DC dream starts to run into reality.

**Tank heating** is just the energy table above — divide by your power and you get time. At 52V with a 1500W DERNORD element:

- Short shower worth of water (5 gal, winter): 1.0 kWh ÷ 1.5 kW = **40 minutes to reheat**
- With a charged tank at 160°F setpoint: stores 50% more energy than a 120°F tank — same 5 gallons now covers a full shower with room to spare, and recharge time drops to ~14 minutes

**Tankless heating** has to deliver all the power in real time, while the water flows. The formula:

> **Power (W) = flow (L/min) × ΔT (°C) × 69.8**

At the sink (2 L/min, ΔT = 45°C winter):
- Power needed: 2 × 45 × 69.8 = **6,282W**

At the shower (8 L/min, 1.5 GPM, ΔT = 45°C):
- Power needed: **~25,000W**

---

## The Voltage/Current Wall

Here's where the 52V DC system runs into a hard limit. Current draw at the sink in winter:

| Voltage | Amps (sink, winter) |
|---|---|
| 52V | 121A |
| 120V | 52A |
| 240V | 26A |

And the mnemonic, which fell out of the math naturally:

> **At 120V → ~50A per sink winter. At 52V → ~120A per sink winter. The voltages and amps swap.**

Shower = 5× the sink flow = **5× the amps**. At 52V that's 600A. At 120V it's 260A. Neither is practical.

This is the fundamental reason tankless shower heating means 240V or gas, full stop. It's not a design choice — it's physics. Low voltage means high current means thick wire that won't fit in the element tube.

To illustrate: a 4500W/52V element needs R = 52²/4500 = **0.6Ω**. A standard 4500W/240V element has R = 12.8Ω. To get 0.6Ω in the same wire length you need ~20× larger cross-section wire — roughly 4–5× the diameter. It won't coil into the tube.

We briefly considered 12 DERNORD elements plumbed in series under the RV floor. Each adds its increment of heat. The image is genuinely funny. We did not pursue this further.

---

## The Tank Wins

The impulse cooktop insight applies perfectly to water heating — buffer slowly, deliver fast — but unlike cooking, water tolerates a much more civilized arrangement. You don't need to deliver all the energy in 10 minutes. You charge the tank over an hour or two, then draw it down in a shower.

At 160°F setpoint (safe for a tank with a TMV to prevent scalding at the tap), a standard 5-gallon RV tank stores **~1,500 Wh** — enough for a proper low-flow shower with headroom. Reheat time from empty at 1500W: about 60 minutes. Reheat from a partial draw (the normal case): 15–20 minutes.

Your existing 5-gallon Suburban tank, wired to your DC bus with a DERNORD 48V/1500W element in place of the stock 120V element, is already the answer.

The 5-gallon-for-dishes claim, meanwhile, remains unexplained by physics. Some mysteries are not worth solving.

---

## The Complete Rules of Thumb

**Energy (stored / tank):**
- 120 Wh/gal summer (70°F inlet, 120°F setpoint)
- 200 Wh/gal winter (40°F inlet, 120°F setpoint)  
- Winter ≈ 1.5× summer
- At 160°F setpoint: 300 Wh/gal — 50% more stored energy, same tank

**For metric users:** 1.16 Wh per liter per °C — multiply by your ΔT.

**Power (tankless):**
- 120V → ~50A per sink, winter
- 52V → ~120A per sink, winter (voltages and amps swap — use your system voltage as the mnemonic)
- Shower = 5× sink flow = 5× power
- Tankless shower requires 240V. Below that, use stored hot water.

**Tank recharge at 52V / 1500W:**
- ~14 minutes per shower-worth of water (from a warm tank)
- ~60 minutes full cold recharge of 5 gallons

---

*All calculations assume resistance heating with COP=1. Heat pump water heaters achieve COP of 3–4× but require different analysis. Summer/winter inlet temperatures are approximate US averages and vary significantly by region and water source depth.*
