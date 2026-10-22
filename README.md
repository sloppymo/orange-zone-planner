# Orange Zone — Job Sheet

Pick who does which job in each work phase of the Orange Zone solo board
game.

## Run it

```
cd /home/sloppymo/Documents/orange-zone-planner
python3 -m http.server 8765
# open http://127.0.0.1:8765/index.html
```

(Opening `index.html` directly in a browser also works. The only outside
request is the IM Fell English and Kalam fonts from Google Fonts; offline
it falls back to system fonts.)

## Using it

The page is drawn as an open ledger: survivors on the left page, their
jobs across the spine on the right (on a phone the two pages stack).
One row per survivor, with their current statuses from the sheet as small
icons (injury, plague, stress, hunger, fatigue; a key at the foot of the
left page names them, and numbers of 25 or more are coloured). Pick a job for Phase 1 and Phase 2 from each
dropdown.

- Each job shows that survivor's grade from the sheet's grade table, as
  effectiveness / suitability, which is the sheet's benefit / cost (`–`
  where the sheet has `--`). It appears in the
  dropdown list and, coloured, on the chosen job. The Forge, Rest and
  missions have no grade.
- Jobs keep their slot limits: a full two-slot job is greyed out, and
  picking a full one-slot job takes the previous worker off it.
- **Workers** counts real jobs against the cap of 5. Rest and missions
  don't count, and going over turns the count red.
- **Power** shows the start and end of each phase: the bike adds 2 first
  (up to the max), and each staffed Forge, Workshop or SIGINT costs 1
  however many people work it. Phase 2 starts where Phase 1 ends. Click the
  starting number to correct it.
- Margin notes under the tally list what the plan as written triggers:
  Look There (+25% resources when Philippe pairs at Water, Hydroponics,
  Logging or Iron), Field Tech (Escobedo at a station under 80 condition),
  Loner (Escobedo sharing a job), Overconfident (Philippe working alone),
  Claustrophobic (Diego resting), a staffed Forge with nothing queued, and
  Ida with no Workshop, SIGINT, Hydroponics or Lookout job all day.
- **The left page is a grade grid.** Each survivor's row has a column per
  graded job (Bike, Water, Workshop, SIGINT, Hydroponics, Logging, Iron,
  Lookout), with effectiveness over suitability (the key beside "Survivor" says so;
  on narrower screens it moves to the key at the foot of the page). Read down a column
  to see who fits that job best. Hovering a column heading shows the
  station's stock, progress and condition. Hovering a cell shows the grade
  and any trait effect of working there.
- Click a cell to put that survivor on the job, and click it again to take
  them off. The ring marks the current job. The grid plans whichever phase
  is highlighted: switch with the Phase 1 / Phase 2 tabs at the top of the
  left page, or click a Phase heading on the right page. Slot rules and the
  dropdowns stay in step. Rest, the Forge and missions have no grade, so
  they stay in the dropdowns.
- **Copy plan** puts the day's plan on the clipboard. **Next day** (the
  folded corner at the bottom right of the page) turns the page, clears
  both phases and resets power to base.

The plan is kept in the browser and reset when a new sheet sync arrives.

## Files

| file       | purpose                                                  |
|------------|----------------------------------------------------------|
| index.html | the job sheet                                            |
| full.html  | the earlier full planner (Auto-suggest, dice, supplies)  |
| state.js   | generated game-state snapshot (DO NOT hand-edit)         |
| sync.py    | regenerates state.js from a sheet export                 |

## Syncing from the spreadsheet

Download the Google Sheet as xlsx into ~/Downloads, then:

```
python3 sync.py            # picks newest "The Orange Zone Sheet*.xlsx"
python3 sync.py path.xlsx  # or explicit file
```

Syncs: day, weather, power, all 6 characters (statuses, base & effective
skills, hydration, traits, conditions), inventory stocks (including wood,
metal, and perishables), station conditions/abundance, SIGINT reveal
progress, Workshop banked progress, Forge queue, and a content hash.
Both pages detect a new export on load (day, filename, or content hash
changed).

---

# The full planner (`full.html`)

Everything below describes `full.html`, which also suggests plans,
records dice and tracks supplies. It keeps its own browser state, separate
from the job sheet, and preserves your settings across syncs (each job's
priority, open/closed and Water Tanks power boost, plus the worker cap and
d100 thresholds).

## Daily flow

1. Open the page. A phase that starts with nobody assigned gets a suggested
   plan straight away. **Suggest plan** reruns it; **Clear** empties the
   phase and keeps it empty, even after a reload.
2. **Who works where** is the board: one row per survivor, one column per
   job in turn order. Each cell shows what that survivor would make there and
   the expected status cost (`FTG +17`). A cost in colour crosses a penalty
   line (25 / 50 / 76), and ☠ marks a death risk. Greener cells score better
   with Auto; orange cells are assigned.
   - Click a cell to assign, and click it again to unassign. A full one-slot
     job swaps the old worker out; a full two-slot job asks you to take
     someone off first. Rest doesn't count against the worker cap.
   - Hover a cell to preview that survivor's status bars after the job. The
     tooltip has roll formulas, d100 thresholds, trait notes, the computed
     and sheet grades, and Auto's score.
   - Survivor name opens the editor (statuses, base skills, hydration, CIGS);
     ✈ marks them away on a mission today.
   - The dial row under the board sets each job's priority: `0` = never
     suggest, `1` = `×1`, `2` = `×2.5`, `3` = `×6`.
3. The **Phase outcome** panel shows power left after the phase, yields
   (food in nights, raw water against the tank, wood, metal, Lab / Forge /
   SIGINT progress), the plague card cancel chance, a **Watch out** list
   (penalty lines, death risks, idle survivors, Ida's MENT/SCI need, Look
   There, weather damage) and **Why this plan** (Auto's breakdown, flagged
   when you change an input after the suggestion).
4. Click a job's column header for its dialog: open/closed, exact priority
   (`0.5` steps), condition, abundance, the Water Tanks power boost, a
   who-fits-best table, and the rules text.
5. **Record the dice** lists what each worker rolls and their accident and
   lucky thresholds. Boxes start at the expected result; type what actually
   happened (Shelter fatigue accepts `14` or `-14`) and press **Apply phase
   N**. Statuses update, power settles, fruit / wood / metal / raw water /
   Lab / Forge / SIGINT are credited, the phase locks, and Phase 2 opens with
   a fresh suggestion. **Undo apply** reverts the last apply.
6. **New day** clears plans, missions and daily flags (`CIGS`) and recharges
   power to base. It asks for a second click if a phase isn't applied yet.

The supplies strip under the top bar is editable. **Settings** holds the
worker cap, the d100 accident and lucky thresholds, the full grade sheet,
and **Reset to sheet** (two clicks).

## How Auto-suggest scores

It tries every legal plan (slots, worker cap, power never below 0 with
the bike resolving first, and both unpowered/powered Water Tanks) and
keeps the highest:

    plan value = Σ prioM(prio) × station value − Σ worker strain
                 + rest value + Ida bonus + power reserve

`1 point` ≈ one good shift. All numbers live in the `VAL` table at the
top of the script in `index.html` — tune them there.

| item | value | notes |
|------|-------|-------|
| fruit | 5 hunger ÷ (survivors × 20/night) × urgency | urgency `×1.5` under 1.5 nights of food, `×1` under 3, `×0.6` otherwise |
| wood / metal | `1/20` · `1/12` per unit | |
| raw water | `1/5` per unit | only up to free tank space (cap − stock − weather water) |
| Lab / Forge progress | `1/40` per point + `0.75` for finishing | capped at remaining need; Forge is 0 when queue is `None` |
| SIGINT | `1/60` per point | never capped — excess carries over |
| Lookout | `1` × chance of at least one cancel | workers combine: `1 − Π(1 − cancel%)` |
| power | not a direct benefit | legality check; leftover power is worth `0.15 × prioM(bike prio)` in Phase 1 (and in Phase 2 when Bike `prio > 1`) |
| Ida (`EASILY BORED`) | `+0.4` at a MENT/SCI station | not in Phase 2 if Phase 1 already had her at one |
| strain per status point | FTG `.01` · HUN `.01` · INJ `.03` · STS `.015` · PLA `.02` | |
| accident station damage | `0.60 × accident chance` | reflects the wood/metal cost of repairing 10–15 station condition damage |
| status band level | `0 / 0.25 / 0.5 / 1 / 100` | for `<25 / 25–49 / 50–75 / >75 / 100+ (death)`; strain = level you end in − level you started in, averaged over exact dice + accident odds |
| rest | recovered fatigue scored by the same strain table | `× prioM(Shelter prio)`; side effects (`Claustrophobic +6 stress`) are not scaled |

Dials (`VAL.prioMult`): **0** = station off; **1** = `×1`; **2** = `×2.5`;
**3** = `×6` its value (interpolated for `0.5` steps).

## Rule interpretations baked in

- Station power cost is **per station**, not per worker (per ruling).
- Philippe's `Look There` (`+25%`) applies to **resources only** (wood, metal,
  fruit, water), not Lab/Forge/SIGINT progress (per ruling).
- Base skills (`max`, from `Plague Data`) are used as the starting point for
  status penalty calculations so penalties are never applied twice.
- Hydroponics condition penalty is `+5` fatigue/step like all other
  stations (sheet text is a typo), and Lookout Station has `+5` fatigue and
  `−1%` cancel chance per 20% condition loss.
- Workshop and SIGINT apply their `−5` luck roll penalty per 20% condition
  loss to accident/luck d100 thresholds alongside hydration.
- Water Tanks accidents split their `+7` worker status damage 50/50 between
  `injury` and `stress` (`chosen at random`).
- Weather `"Any worker"` penalties apply to working stations, not survivors
  resting at The Shelter.
- Status bands: `25–49 = −10%`, `50–75 = −20%`, `>75 = −40%` (`75` is `20%`,
  `76+` is `40%`).
- Station condition penalties use `floor((100−cond)/20)` steps.
- Bike power is generated before lower stations run, so it can fund
  same-phase power costs.
