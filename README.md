# Orange Zone — Work Phase Planner

Speeds up the "assign survivors to workstations" step of the Orange Zone
solo board game. Replaces the mental math + hand-entered grade matrix in
the tracking spreadsheet.

## Run it

```
cd /home/sloppymo/Documents/orange-zone-planner
python3 -m http.server 8765
# open http://127.0.0.1:8765/index.html
```

(Opening `index.html` directly in a browser also works — it's a single
file with no dependencies.)

## Files

| file       | purpose                                              |
|------------|------------------------------------------------------|
| index.html | the whole app                                        |
| state.js   | generated game-state snapshot (DO NOT hand-edit)     |
| sync.py    | regenerates state.js from a sheet export             |

## Syncing from the spreadsheet

Download the Google Sheet as xlsx into ~/Downloads, then:

```
python3 sync.py            # picks newest "The Orange Zone Sheet*.xlsx"
python3 sync.py path.xlsx  # or explicit file
```

Syncs: day, weather, power, all 6 characters (statuses, skills, hydration,
traits, conditions), inventory stocks, station conditions/abundance,
SIGINT reveal progress, Workshop banked progress, Forge queue.

A new sheet export is auto-detected on page load (day or filename changed)
and stale browser state is discarded — no manual Reset needed.

## Daily flow

1. Check the **needs strip** (food runway, water stock, Lab/SIGINT
   remaining, Forge queue) and set station **prio** dials (0 = off,
   1 = normal, 2 = ×4 value, 3 = ×9 value — see below).
2. **Auto-suggest** for a legal-by-construction plan, or click a
   survivor then a station slot. Click a filled slot to swap. The totals
   bar shows Auto's value breakdown until you change the plan.
3. Preview lines show roll formulas (`1d10+9`), expected values,
   accident/luck % (hydration-adjusted), trait notes, and threshold
   warnings (`fatigue 17→~32 ⚠−10%`).
4. Roll dice, enter actual results in **Apply rolled results** (statuses
   plus the OUT column) → statuses update, power is settled (bike
   generation minus station costs), fruit / raw water / Lab banked /
   SIGINT progress are credited, and the phase advances. Wood and metal
   are shown but not credited — record them in the sheet. An applied
   phase is locked (no second Apply).
5. Repeat for Work Phase 2. **New day** clears plans/mission flags and
   recharges power to base.

## How Auto-suggest scores

It tries every legal plan (slots, worker cap, power never below 0 with
the bike resolving first) and keeps the highest:

    plan value = Σ prio² × station value − Σ worker strain
                 + rest value + Ida bonus + Phase-1 power reserve

1 point ≈ one good shift. All numbers live in the `VAL` table at the
top of the script in index.html — tune them there.

| item | value | notes |
|------|-------|-------|
| fruit | 5 hunger ÷ (survivors × 20/night) × urgency | urgency ×1.5 under 1.5 nights of food, ×1 under 3, ×0.6 otherwise |
| wood / metal | 1/20 · 1/12 per unit | |
| raw water | 1/5 per unit | only up to free tank space (cap − stock − weather water) |
| Lab progress | 1/40 per point + 0.75 for finishing | capped at what the Lab still needs; Forge is 0 with no queue |
| SIGINT | 1/60 per point | never capped — excess carries over |
| Lookout | 1 × chance of at least one cancel | workers combine: 1 − Π(1 − PER%) |
| power | not a benefit | legality only; in Phase 1 each power left is worth 0.15 × bike prio² |
| Ida (EASILY BORED) | +0.4 at a MENT/SCI station | not in Phase 2 if Phase 1 already had her at one |
| strain per status point | FTG .01 · HUN .01 · INJ .03 · STS .015 · PLA .02 | |
| status band level | 0 / 0.25 / 0.5 / 1 / 10 | for <25 / 25–49 / 50–75 / >75 / 100+; strain = level you end in − level you started in (0→50 costs 0.5), averaged over the exact dice and accident odds, so a 30% chance of crossing costs 30% of it |
| rest | recovered fatigue scored by the same strain table | × Shelter prio²; side effects (Claustrophobic +6 stress) are not scaled |

Dials: **0** = station off; 1 = normal; 2 = ×4; 3 = ×9 its value
(a station with nothing left to gain, e.g. a finished Lab, stays at 0).
Shelter prio scales the value of rest; Bike prio scales the Phase-1
power reserve.

## Rule interpretations baked in

- Station power cost is **per station**, not per worker (per ruling).
- Philippe's Look There (+25%) applies to **resources only** (wood, metal,
  fruit, water), not Lab/SIGINT progress (per ruling).
- Hydroponics condition penalty is +5 fatigue/step like all other
  stations (sheet text is a typo).
- Hydration applies to accident/luck d100 thresholds.
- Workshop progress beyond what the Lab still needs is treated as wasted
  (nothing queued after it); "stores up to 50 progress" isn't modeled.
- SIGINT excess carries into the next scan, and each completed scan adds
  5 to the requirement (Workstations tab).
- Status bands: 25–49 = −10%, 50–75 = −20%, >75 = −40% (75 is still 20%).
- Station condition penalties use `floor((100−cond)/20)` steps.
- All skill math uses effective (status-penalized) skill.
- Bike power is generated before lower stations run, so it can fund
  same-phase power costs.
