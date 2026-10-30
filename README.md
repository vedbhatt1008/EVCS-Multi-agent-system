# EV Charging Simulation (NetLogo)

A simulation of **electric-vehicle (EV) charging stations that share one power connection (a transformer)**.
It tests whether smart rules (dynamic prices, a "fast-only" mode, and power sharing between stations) can reduce overload and keep EV drivers happy, compared with a simple "do nothing" approach.

- **File:** `Simulation.nlogox`
- **Software:** NetLogo 7.0.4 (free: https://ccl.northwestern.edu/netlogo/)
- **Simulated time:** 7 days, 1 tick = 1 minute (10,080 ticks)

---

## 1. The big idea in plain words

Imagine a car park with **4 charging stations**. All four get their electricity from **one transformer** that can only deliver **820 kW** in total.

If too many EVs plug in at once, the stations want more power than the transformer can give. This is called **overload**.

The simulation compares two worlds:

1. **Uncontrolled** – every EV goes to its nearest station and charges. No smart rules.
2. **Smart methods** – stations change behavior when busy, show prices that rise when busy, and lend unused power to each other. Drivers choose stations by price.

At the end we measure: how often was there overload, how long did charging take, and how many EVs got charged enough?

---

## 2. What you see on the screen

| Symbol | Meaning |
|---|---|
| Yellow square (center) | The **transformer** (turns **red** when requested power is above 820 kW). Label shows `delivered/820 kW`. |
| House shapes (4 corners) | The **charging stations** S1–S4. Sky-blue = normal mode, violet = fast-only mode. |
| Green squares | **Slow plugs** (5 per station, 11 kW) |
| Orange squares | **Fast plugs** (5 per station, 30 kW). Slow plugs also turn orange when they run in fast mode. |
| Car shapes | **EVs**. **Red** = urgent driver, **white** = non-urgent driver. |
| Cars near a station | EVs **waiting in the queue** |
| Cars on a plug | EVs **charging** |

**Station label example:** `S1 R=0.9 FAST 280/205 L=205`
- `R=0.9` → congestion ratio (see below)
- `FAST` / `norm` → current mode
- `280/205` → requested kW / delivered kW
- `L=205` → power allocation of this station (kW)

---

## 3. Interface (buttons, slider, chooser, monitors)

| Item | What it does |
|---|---|
| **scenario** (chooser) | Selects one of 5 scenarios (see section 4). |
| **ev-arrivals-per-hour** (slider) | How many EVs arrive per hour on average (base rate). Range 5–20 in the interface. |
| **setup** | Clears everything and builds a fresh model with a random seed. |
| **go** | Runs the simulation continuously, one minute per tick (stops at the end of the week). |
| **go-fast** | Same as `go`, but draws the screen only once per simulated hour, so it is much faster. |
| **run-simulation** | One click: does `setup` and runs the whole week at once. |
| **run-protocol** | Runs **all 5 scenarios × 30 seeds** automatically and writes `protocol_results.csv` (next to the model file). |
| **r-display** (monitor) | Shows R and the mode (norm/FAST) of every station. |
| **fulfilled-display** (monitor) | Shows `served of total EVs (percent)`. |
| **Price vs R** (plot) | Shows how the fast price depends on R (see section 6). Colored dots show the live position of each station. |

### Quick start
1. Open `Simulation.nlogox` in NetLogo.
2. Pick a **scenario**.
3. Click **setup**, then **go-fast**.
4. Watch the stations, the monitors and the plot. The run stops after 7 days.

---

## 4. The five scenarios

| Scenario | Fast-only mode | Local Grid (power sharing) | Price-based choice | Price type | Parking time |
|---|---|---|---|---|---|
| `uncontrolled` | no | no | no (nearest station) | fixed prices | 60–600 min |
| `method1` | yes | yes | yes | **step** | 60–600 min |
| `method2` | yes | yes | yes | **ramp** | 60–600 min |
| `method1-capped` | yes | yes | yes | step | shorter: 60–540 min |
| `method2-capped` | yes | yes | yes | ramp | shorter: 60–540 min |

"Capped" means EVs are allowed to park for a shorter time, so charging is more urgent.

---

## 5. How the model works (step by step)

### 5.1 The setup
- **1 transformer** with capacity **820 kW**.
- **4 stations** in the corners of the map. Each begins with an equal allocation of 820 / 4 = **205 kW** (this number is only a starting point, not a hard limit).
- Each station has **10 plugs**: 5 slow (11 kW) and 5 fast (30 kW).

### 5.2 Arriving EVs
All EVs of the week are **pre-generated** before the run starts. This means that **the same seed gives exactly the same EVs in every scenario**, so scenarios can be compared fairly.

How many EVs arrive each minute depends on the slider and on the time:

| Time | Factor |
|---|---|
| Night (00:00–06:00) | × 0.5 |
| Daytime | × 1 |
| Evening peak (17:00–22:00) | × 1.5 |
| Days 6–7 (holiday weekend) | additional × 1.2 |

Each EV gets random properties:

| Property | Value |
|---|---|
| Battery size | 40–80 kWh |
| Charge on arrival | 10–50 % |
| Parking time | 60–600 min (capped scenarios: 60–540 min) |
| Urgent? | 30 % of EVs are urgent |
| Price limit (highest price the driver accepts) | Non-urgent: 0.40–0.70 EUR/kWh. Urgent: 0.70–1.00 EUR/kWh |
| Arrival location | 60 % of arrivals appear near a "busy" station, 40 % anywhere on the map |

**Busy station ("surge"):** every 3 hours, one random station becomes the busy one. 60 % of arriving EVs appear near it. This creates uneven traffic, which is exactly the situation where power sharing helps.

An EV is **served** when it reaches **80 % of its battery** before it leaves.

### 5.3 Choosing a station
- **Uncontrolled:** the EV goes to the **nearest** station.
- **Price scenarios:** the EV looks at the price shown by every station and ignores those above its own price limit. Among the rest it takes the **cheapest**; ties are broken by **nearest**, then by **lowest R**.
- If **no station is cheap enough**, the EV leaves immediately ("balked") and counts as **unserved**.
- Once the EV has chosen, it **never changes its mind**.

### 5.4 Queue and plugs
- The EV joins the station's queue (first come, first served).
- **Urgent** EVs prefer a **fast** plug, **non-urgent** EVs prefer a **slow** plug. If the preferred type is full, the EV takes any free plug.
- If no plug is free, the EV waits. If its parking time runs out while waiting, it leaves unserved.

### 5.5 Congestion ratio R
Each station calculates

```
R = (EVs charging + EVs waiting) / number of plugs (10)
```

R = 0.5 means half full. R = 1.0 means every plug is used (or more EVs are waiting).

### 5.6 Modes: normal and fast-only
- **Normal mode:** slow plugs run at 11 kW, fast plugs at 30 kW.
- **Fast-only mode:** when **R ≥ 0.8**, the station switches to fast-only. Now **all plugs run at 30 kW**, so every EV charges faster and plugs free up sooner.
- When R drops below 0.8, the station goes back to normal.
- **Minimum mode duration:** after any switch, the station must stay in its new mode for at least **10 minutes** (in both directions). This prevents rapid flickering.
- This only happens in `method1/2` scenarios. `uncontrolled` is always normal.

### 5.7 Prices
| Situation | Price (EUR/kWh) |
|---|---|
| Normal mode, slow charging | 0.35 |
| Normal mode, fast charging (urgent EVs see this) | 0.60 |
| Fast-only mode, **Method 1 (step)** | jumps straight to **0.80** |
| Fast-only mode, **Method 2 (ramp)** | rises smoothly from 0.60 (at R = 0.8) to 0.80 (at R = 1.0) |

Method 2 formula: `price = 0.60 + (0.80 − 0.60) × x`, where `x = (R − 0.8) / (1.0 − 0.8)`, limited to between 0 and 1.

The price an EV sees when choosing is the price it **pays for the whole session** (the "session price").

### 5.8 Power delivery
Every minute, each station:
1. **Demand D** = sum of the maximum power of all occupied plugs.
2. **Available power** = the smaller of D and the station's allocation **L**.
3. This power is **shared equally** between the charging EVs, but no EV gets more than its plug's maximum.
4. Each EV gains `rate / 60` kWh per minute.

### 5.9 Local Grid (power sharing between stations)
The sum of all allocations is always **820 kW**. The Local Grid only **moves** power between stations; it never creates new power.

Every minute (at the end of the tick, so it is used from the **next** tick on):
1. **Return loans:** a station that is back in normal mode gives borrowed power back to its donor.
2. **Request power:** a station in fast-only mode asks for what it is missing (`need = demand − allocation`). The donor is the station with the **largest unused headroom** (`allocation − demand`). One donor per request. Stations that currently hold borrowed power cannot be donors.
3. A **loan ledger** (`loan-matrix`) remembers who borrowed how much from whom, so it can be returned correctly.

The loan **persists** until the borrower returns to normal mode.

### 5.10 Safety checks (invariants)
Every tick the model checks that:
- the sum of all allocations is exactly 820 kW,
- no allocation is negative,
- no station delivers more than its allocation.

If one check fails, the run stops with an error message. (This is a built-in test that the logic is correct.)

### 5.11 Leaving
- **Served:** EV reached 80 % → its cost and times are recorded, the plug is freed, the EV disappears.
- **Unserved:** parking time ran out before 80 % → it leaves (also from the queue).
- At the end of the week, EVs still in the system count as unserved.

### 5.12 Order of one minute (tick)
1. Update R of every station
2. Update mode
3. Update price
4. New EVs arrive and choose a station
5. Queued EVs get free plugs
6. Power is delivered and shared
7. Invariants are checked
8. EVs charge
9. EVs leave
10. Local Grid adjusts allocations (for the next tick)
11. Update R, record metrics, update labels and plot

---

## 6. The "Price vs R" plot

- **Gray line:** slow price in normal mode (0.35)
- **Red line:** Method 1 – flat at 0.60, then jumps to 0.80 at R = 0.8
- **Blue line:** Method 2 – flat at 0.60, then ramps up to 0.80 at R = 1.0
- **Colored dots:** the current R and price of stations S1–S4

If you delete this plot, set `price-plot-wanted?` to `false` in `set-parameters`, or the model will give an error.

---

## 7. Results (what is measured)

### Main outputs
| Output | Meaning |
|---|---|
| **Overload minutes / episodes** | Minutes in which total requested demand **equals** 820 kW. Consecutive minutes count as one episode. |
| **Average charging time** | Minutes from the **start of charging** until 80 %, for served EVs. |
| **Service rate** | Percent of EVs that reached 80 % before leaving (balked EVs count as unserved). |

### Extra outputs
`overload_above_minutes` (demand above 820 kW), `below80_evs`, `below80_share_pct`, `avg_time_from_arrival` (includes waiting), `avg_wait_min`, `peak_demand_kw`, `balked_evs`, `avg_price_eur_kwh`, `unserved_kwh`, `unmet_kwh`, `fast_only_station_minutes`, `borrowed_kwh`.

---

## 8. Running the full experiment

Click **run-protocol**. It runs every scenario for seeds 1–30 and creates **`protocol_results.csv`** with one row per run. The live plot is switched off during this to make it faster. Because the same seed means the same EVs, differences between scenarios come only from the rules, not from luck.

---

## 9. Key parameters (in `set-parameters`)

| Parameter | Value | Meaning |
|---|---|---|
| `c-main` | 820 kW | Transformer capacity |
| `n-slow`, `n-fast` | 5, 5 | Plugs per station |
| `w-slow`, `w-fast` | 11 kW, 30 kW | Plug power |
| `r-high` | 0.8 | R at which fast-only mode starts |
| `r-max` | 1.0 | R at which Method 2 reaches the max price |
| `min-mode-duration` | 10 min | Minimum time in a mode |
| `p-slow`, `p-fast`, `p-cap` | 0.35, 0.60, 0.80 EUR/kWh | Prices |
| `p-urgent` | 0.3 | Share of urgent EVs |
| `required-charge` | 0.8 | Target = 80 % of the battery |
| `sim-length` | 10080 | 7 days in minutes |
| `surge-minutes` | 180 | How long a station stays "busy" |
| `surge-share` | 0.6 | Share of EVs appearing near the busy station |

---

## 10. Things to try

- Compare `uncontrolled` with `method1` and `method2` at the same arrival rate.
- Increase the arrival slider and watch overload and service rate.
- Watch the transformer turn red and see which scenario spends less time red.
- Watch the **L=** values on the stations change as power moves to busy stations.
- Compare the normal and `capped` scenarios to see the effect of shorter parking.

---

## 11. Notes about the code

- Comments at the top of the code mention a slider range of 10–40 and 5 seeds, but the **actual** values are: slider **5–20** (`min-arrivals` / `max-arrivals`), and `run-protocol` uses **30 seeds**. The code values are what counts.
- The code mentions "report Sections/Equations"; these refer to the accompanying written report.
- The `info` tab inside the NetLogo file is still the empty template. You can paste this README there if you like.
