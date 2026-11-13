# Napoli Pizza Calculator: technical writeup

This document records how the app works: what it does, how every calculation works, where each number comes from, how the page is structured, how it's deployed and tested, and what is still open. It's written so a new working session can continue without the original conversations.

- **Live site:** https://manuelvrhovac.github.io/napoli-dough-calculator/
- **Repository:** https://github.com/manuelvrhovac/napoli-dough-calculator (branch `main`, published by GitHub Pages from `main`)
- **Local checkout:** `/Users/Manuel/Documents/NapoliDoughCalc`
- **Main file:** `index.html` (about 1,120 lines, single file, no build step, no dependencies)
- **Images:** `images/PizzaCanotto.webp`, `images/PizzaVerace.webp` (2048 px wide, transparent background)
- **Source documents:** `docs/sources/` (see [Source documents](#2-source-documents))
- **Last design commit when this was written:** `1086c52 Redesign configurator into collapsible sections`

---

## Contents

1. [What the app is](#1-what-the-app-is)
2. [Source documents](#2-source-documents)
3. [How the project evolved](#3-how-the-project-evolved)
4. [Architecture](#4-architecture)
5. [State model and persistence](#5-state-model-and-persistence)
6. [The calculation engine](#6-the-calculation-engine)
7. [Fermentation scheduling and the bake-time picker](#7-fermentation-scheduling-and-the-bake-time-picker)
8. [The Dough check](#8-the-dough-check)
9. [The recipe timeline](#9-the-recipe-timeline)
10. [User interface](#10-user-interface)
11. [Visual design](#11-visual-design)
12. [Icons and licenses](#12-icons-and-licenses)
13. [Deployment](#13-deployment)
14. [Testing approach](#14-testing-approach)
15. [Estimates and assumptions to verify](#15-estimates-and-assumptions-to-verify)
16. [Known issues and gotchas](#16-known-issues-and-gotchas)
17. [Backlog and ideas](#17-backlog-and-ideas)
18. [Reference figures](#18-reference-figures)

---

## 1. What the app is

A mobile-first, single-page calculator for hand-mixed Neapolitan pizza dough, built around the owner's setup:

- **Oven:** Witt Etna Fermo (gas, fixed stone, no built-in thermometer)
- **Mixing:** by hand
- **Flours:** Caputo Pizzeria (W 260–280) and a strong W ~380 flour for blends. The app itself no longer names flours; it works with W values only.
- **Yeast:** fresh yeast or instant dry (7 g sachets)
- **Typical batches:** 12 balls for a party of 10, or 3 balls for a small dinner

You choose a style, hydration, ball size, start and bake times, flour strength and fold sets. The app gives you:

1. Three collapsible settings sections, each with an icon summary when closed
2. A collapsible **Dough check** that flags risky combinations
3. An **Ingredients** table with the ball-count stepper, split into bowls of up to 6 balls
4. A **Timeline** with real clock times, where every step shows its exact ingredients as pills and the current step's dot flashes

Everything is client-side. Choices persist in `localStorage`.

---

## 2. Source documents

Two documents drive every number. Copies are in `docs/sources/`.

| File | Role |
|---|---|
| `Canotto Pizza Dough — Witt Fermo (24 36 48 h).md` | The first base: a 12-ball, 48 h party recipe (75% Caputo Pizzeria + 25% Manitoba, 66% hydration, 270 g balls, 0.05% instant yeast). Used for version 1. |
| `neapolitan_pizza_knowledge.md` | The much larger knowledge file (about 1,340 lines): a 20-recipe audit, STG and AVPN rules, flour W tables, hydration ladder, yeast dose by schedule, kneading, fermentation, balling, appretto, Witt Fermo baking, troubleshooting and corrected recipes. **It supersedes the first document wherever they disagree.** |

Key corrections the knowledge file made to the first document:

- **Yeast for 48 h cold** was raised from 0.05% instant (0.15% fresh) to **0.10% instant (0.30% fresh)**, because 0.05% sits below every comparable recipe.
- **Weigh yeast directly** on a 0.01 g scale and whisk instant yeast into the flour. The solution method is the fallback, and dry yeast settles in it within minutes.
- **Kneading** is two-stage: about 10 min, rest 10 min, then 5–10 min more, followed by 2–3 fold sets.
- **Warm time before the fridge** is about 2 h.
- **Balling** happens 5–6 h before baking, and the oven is preheated for 45 min.

The first document also contained an internal inconsistency: its formula said 66% hydration while its method text said 68% in places. The app uses 66%.

---

## 3. How the project evolved

The app was built over many rounds of feedback. Knowing the order helps explain some code shapes.

| # | Change |
|---|---|
| 1 | Basic calculator from the canotto document, published as a private Claude artifact. |
| 2 | Single column for mobile, with a beige, orange and pastel red palette and gradients. |
| 3 | **Auto** fermentation from start and bake times, and hydration as four segmented choices. |
| 4 | Preset choices everywhere: ball weights, five kitchen temperatures, salt choices, and a flour mixer with name, W slider and share. |
| 5 | Flour pick list, dated timeline, steps, a **Now** button rounding up to the next 15 min, a green secondary color, and the rename to "Napoli Pizza Calculator". |
| 6 | First Dough check, a rule-based warning system marked work in progress. |
| 7 | **Rebuilt on the knowledge file:** new yeast curve, Canotto and Verace styles, an 8 h same-day option, fridge temperature, yeast weighing method, two-stage kneading and updated check rules. |
| 8 | 22–51 h tolerance, side-by-side Start and End, a fold-set choice, steps merged into the timeline with ingredient pills, a flashing current-step dot. |
| 9 | Moved into a git repo, given a standalone HTML head, a fun README, and published on GitHub Pages. |
| 10 | Dough check audited against the knowledge file and rewritten. |
| 11 | Collapsible configurator with a summary button, Start and End as inputs with suggested hours, fresh yeast as the default, a yeast test, a confirmed Reset, an iOS time-field fix. |
| 12 | Icons, Apple's system font, ±5 min start shifters. |
| 13 | Pure white theme without dark mode, colored icons, darker red, time inline with the step title. |
| 14 | **Big redesign (commit `1086c52`):** style photos and an arched style selector; flour mixer replaced by W steppers with an optional blend; three collapsible sections instead of one; a meal-time bake picker; a fermentation graph; clearer Dough check texts in their own section; ball count moved to Ingredients; many layout refinements. See sections 7–11. |

The private Claude artifact (`https://claude.ai/artifact/Mgn9k5pry6UYXVMF3D4BB9`) stopped being updated after round 8. **The repository is the only up-to-date copy.**

---

## 4. Architecture

One HTML file with inline CSS and JS. There's no framework, bundler or network dependency at runtime (the two webp images are local files).

```
index.html
├── <head>: charset, viewport (viewport-fit=cover), base reset, page <style>
├── <svg> sprite: 17 <symbol id="i-…"> icons + the arcGrad gradient
├── <main class="page">
│   ├── header.top: title + Reset link (with inline confirm)
│   ├── section#config
│   │   ├── details#secStyle   Pizza style (arch selector, photo, ? tip), Hydration, Grams per ball
│   │   ├── details#secFerm    Start/End, bake-time picker, fermentation graph, Flour strength (+ blend), Fold sets
│   │   ├── details#secOther   Salt, Kitchen temp, Fridge temp, Yeast, yeast weighing
│   │   └── details#secCheck   Dough check findings
│   ├── section Ingredients (heading + ball stepper, table, note)
│   └── section Timeline
└── <script>
    ├── /* CALC-START */ … /* CALC-END */   pure logic, no DOM
    ├── data: STYLES, HYD_WHY, PRESETS, MEALS
    ├── state load, migration and validation
    ├── date helpers, renderers (presets, graph, check, timeline, render)
    └── event delegation (click, input, change, toggle, keydown)
```

### The pure-logic block

Everything between `/* CALC-START */` and `/* CALC-END */` is pure: no DOM, no globals outside the block, so it can be pulled out and tested in Node:

```bash
sed -n '/CALC-START/,/CALC-END/p' index.html > "$TMPDIR/calc.js"
```

It contains: `interp`, `roundTo`, `FRESH_CURVE`, `tolerant`, `schedule`, `toMin`, `autoHours`, `TEMP_FACTOR`, `MAX_PER_BOWL`, `defaultFlours`, `DEFAULTS`, `shares`, `blendW`, `weigh`, `calculate`, `W_WINDOW`, `BLEND_STEPS` and `checkDough`.

### Rendering model

- `render()` recomputes everything from `state` and writes the DOM, then saves to `localStorage`. It's called after every change.
- `renderStyle()` rebuilds the ball and hydration choices and swaps the style photo, hint text and thumbnail.
- `renderPresets(h)` rebuilds the bake-time picker list (`#presetList`) and the End button text.
- `renderFtl(s, H, sch)` draws the fermentation graph and the mini graph in the collapsed summary.
- `renderCheck(...)` fills the Dough check section and its collapsed summary.
- All interaction uses **one delegated `click` listener**, plus `change` (time inputs), a capturing `toggle` listener (section open state) and `keydown` (Escape closes the picker).
- Segmented choices are `<div class="seg" id="KEY">` with `<button data-v="VALUE">`. The generic handler writes `state[KEY] = value`, parsed as a number when numeric. Selected state is `aria-pressed="true"`, set in `render()`. The style arch uses the same mechanism (its buttons are children of `#style`).
- `[hidden]` is forced to `display: none !important` by the base reset, so visibility is toggled with `el.hidden`.

---

## 5. State model and persistence

```js
DEFAULTS = {
  style: "canotto", balls: 12, ball: 270, hyd: 66, salt: 2.8, temp: 21, fridge: 4,
  yeast: "fresh", weigh: "scale", fold: 2,
  startTime: "17:00", endDay: 2, endTime: "17:00"
}
// plus: startDate ("YYYY-MM-DD", today), flours ([{ w: 300 }]), open ({ style, ferm, other, check })
```

| Field | Meaning |
|---|---|
| `style` | `"canotto"` or `"verace"` |
| `balls` | 1–15 (stepper in the Ingredients header); at most 3 bowls |
| `ball` | grams per ball, from the style's list |
| `hyd`, `salt` | percentages of flour |
| `temp` | kitchen °C: 19, 21, 23, 25 or 27 |
| `fridge` | `3` (meaning 2–3 °C) or `4` (meaning 4–5 °C) |
| `yeast` | `"fresh"` or `"dry"` (instant) |
| `weigh` | `"scale"` or `"solution"` (dry yeast only) |
| `fold` | 0 or 2 fold sets (3 was removed; the timeline code still handles it) |
| `startDate`, `startTime` | when mixing starts |
| `endDay`, `endTime` | bake day as 0, 1 or 2 days after `startDate`, and the time |
| `flours` | `[{ w }]` or `[{ w }, { w, share }]`. At most 2. The first takes the rest of the share. |
| `open` | which collapsible sections are open; default all closed |

**Fermentation hours are derived, never stored:** `autoHours(s) = (endDay·1440 + endTime − startTime) / 60`.

### Persistence

- Key `canotto-calc-v1` in `localStorage`, written at the end of every `render()` and wrapped in try/catch.
- **The start date is stored**, so a saved plan keeps its real dates. If `startDate` isn't today, a hint shows the saved start.
- Data is per browser and per device. iOS Safari can clear script-written storage after 7 days without a visit, and Private Browsing keeps nothing.
- The origin `manuelvrhovac.github.io` is shared with other Pages projects on that account; the key name keeps it separate.

### Migration and validation on load

Old saved shapes are repaired, so users never see a broken page:

- `ball`, `hyd`, `salt` and `temp` snap to the **nearest** allowed value for the current style.
- An old fixed `hours` is converted to an end time with `setEnd()`, then deleted.
- `cfgOpen` (the old whole-configurator toggle) is deleted; `open` defaults to `{}`.
- `fold` not in `[0, 2]` becomes 2.
- `flours` is cut to 2 entries, names are dropped, W is clamped to 180–420 and rounded to 10, and the second flour's share is clamped to 5–50 and rounded to 5.

### Style change resets

Switching style (not re-tapping the current one) resets hydration and ball weight to the style's defaults, sets `flours` to the style's default (`[{ w: 300 }]` for Canotto, `[{ w: 270 }]` for Verace), sets folds to 2, and starts a fresh **24 h** plan from now rounded up to the next quarter hour. Salt, temperatures, yeast and ball count are left alone. **Reset** (top right) restores `DEFAULTS` but keeps which sections are open.

---

## 6. The calculation engine

### 6.1 Scaling (dough factor)

From the knowledge file (section 3.2):

```
total dough  = balls × ball weight
dough factor = 1 + hyd/100 + salt/100 + yeast%/100      (instant % if dry, fresh % if fresh)
flour        = total dough ÷ dough factor
```

- **Bowls (by flour weight, for hand kneading):** up to 15 balls and 3 bowls. `fit(per)` finds the fewest bowls whose largest bowl stays at or under `per` grams of flour. "Fewer" = `fit(1300)` (never over 1,300 g, so 8 × 270 g still fits one bowl), "more" = `fit(1000)` (comfortable). When they differ, the Ingredients section offers the choice (`state.bowlPref`: `"fewer"` or `"more"`); the default aims for about 1,100 g per bowl. Examples: 7 × 180 g one bowl; 6 × 290 g and 7 × 250 g one bowl by default (choice 1/2); 7 × 270 g two by default; 12 × 270 g two of 6; 12 × 290 g two (choice 2/3); 15 × 290 g three of 5. Each extra bowl is balled 30 min after the previous one.
- **Flour per bowl** is rounded to the nearest **10 g**: `roundTo(bowlBalls × ball / factor, 10)`. Total flour is the sum of the bowls.
- **Per bowl, through `weigh()`:**
  - Second flour: `round(flour × share%)`; the first flour is the remainder.
  - Water `round(flour × hyd%)`, salt `round(flour × salt%)`.
  - Yeast: `flour × fresh% / 100`, divided by 3 for dry yeast (the AVPN 1:3 ratio).
  - **Held-back water:** `clamp(roundTo(water × 5%, 5), 5, 30)`.
  - **Solution** (dry yeast with the solution method): a 7 g sachet made up to 70 g with water. Solution amount is `max(0.5, roundTo(yeast × 10, 0.5))` g.
  - **Plain water in the bowl** = water − solution − held-back water.
- **Totals:** water and salt are recomputed from total flour. Because each ingredient is rounded first, the dough total in the note can be a gram off `balls × ball` (e.g. 2999 g for 12 × 250 g).
- **Flour names** in the table and timeline: `"Flour"` for one flour, `"Flour W 270"` / `"Flour W 380"` for a blend.

**Check:** 12 × 270 g at 66% and 2.8% gives 1,920 g flour, 1,267 g water and 54 g salt, matching both documents.

### 6.2 Yeast dose

**Fresh-equivalent % of flour, by fermentation hours** (`FRESH_CURVE`, from knowledge file section 17.6), linearly interpolated and clamped at the ends:

| Hours | Fresh % | Instant % (÷3) | Source |
|---|---|---|---|
| ≤ 8 | 0.40 | 0.133 | Recipe C, same day |
| 24 | 0.24 | 0.08 | 24 h row (0.20–0.25) |
| 36 | 0.27 | 0.09 | 36 h row (0.25–0.30) |
| ≥ 48 | 0.30 | 0.10 | 48 h row and Recipe A |

Longer cold ferments use slightly **more** yeast than 24 h: fridge hours count for little, so the yeast does most of its work in the warm windows at each end.

**Multipliers:** kitchen temperature (`TEMP_FACTOR`: 19 °C ×1.2, 21 °C ×1.07, 23 °C ×0.93, 25 °C ×0.8, 27 °C ×0.7) and a cold fridge (2–3 °C) on cold schedules ×1.3.

**Tolerance:** `tolerant(h)` maps 22–24 h to 24 h and 48–51 h to 48 h for yeast and flour windows.

### 6.3 Display rounding

- Instant yeast under 3 g shows 2 decimals; other amounts under 10 g show 1 decimal; larger amounts are whole grams.
- The yeast row shows the effective dose, such as `0.32%`.

### 6.4 Flour strength

- **Single flour:** one W stepper, steps of 10, range W 180–420.
- **Blend:** two W steppers plus a share stepper for the second flour (steps of 5, range 5–50%). `blendW` is the share-weighted average.
- **Recommended range** shown under the heading: `interp(tolerant(h), W_WINDOW)`, the same window the Dough check uses.

---

## 7. Fermentation scheduling and the bake-time picker

`schedule(h)` returns `{ cold, fridgeAt, ballAt, rise, fresh }`.

| Hours | Type | Plan |
|---|---|---|
| under 10 | Room temperature only | Bulk until `ballAt`, then a final rise. From 6 h, `rise = h − 2.5`. Under 6 h, `rise = h / 2`. |
| 10 and up | Cold bulk | 2 h warm, fridge, ball at `h − rise`, where `rise` is 5 h at 24 h, 5.5 h at 36 h and later. |

- The **minimum** is 4 h; under that the timeline, graph and check show a prompt instead.
- **9–21 h** uses yeast and timings interpolated between the 8 h and 24 h schedules, without a warning.
- **Over 51 h** works but gets a warning about very strong flour.

### Start

A time input plus a small green **Use current time** link (rounds up to the next 15 min, `nextQuarter`). Changing the start by hand resets `startDate` to today. The ±5 min shifters remain only in the Timeline header (`shiftStart`, which keeps the bake time fixed across midnight).

### End and the bake-time picker

End is a single button styled like the Start field, showing the bake time with the day ("Tomorrow · Sat") underneath. Tapping it opens `#presetMenu`, an overlay with a deep shadow and a light page dim:

- **Tomorrow** and **Day after** blocks, each with three rows of chips:
  - Lunch 11:00–14:00 (yellow, sun icon), Midday 15:00–18:00 (orange, sunset icon), Dinner 19:00–22:00 (indigo, moon icon)
  - each chip shows the resulting hours (`~23 h`) and is disabled ("too soon") under 4 h; a chip sets both `endDay` and `endTime` (`data-bake="1:19:00"`)
- **Or pick a fixed duration:** plain 24 h / 36 h / 48 h chips (`data-preset`), which call `setEnd(hours)`.
- **Or set your own time:** the Today / Tomorrow / Day after segment and the native time input.

The menu closes on a chip choice, a click outside it or Escape; clicks inside it (the custom controls) keep it open. `endDay` is capped at 2, so ferments over about 72 h can't be entered.

### Fermentation graph

`renderFtl` draws a to-scale bar under Start/End:

- Phases: mixing (deep amber, 0–1 h), warm rest (light amber, 1–2 h), fridge (blue), final rise (amber to red). Same-day plans show "Bulk" instead of the fridge.
- Nights (21:00–07:00) are a navy overlay with a small ☾; midnights are thin white lines with the day name above.
- Below: start time on the left, bake time with a flame on the right, and the fridge-in/fridge-out times in the same row, positioned under their ticks and nudged with `clamp(64px, X%, calc(100% − 78px))` to avoid the end labels. Same-day plans show the balling time instead.
- A 10 px label-free copy of the bar (`#ftlMini`) shows in the collapsed Fermentation summary.

---

## 8. The Dough check

`checkDough({ hours, blend, hyd, strongShare, style, fold, temp, fridge })` returns `{ level, findings[], minW, maxW }`. Each finding has `level`, `area`, `title`, `why` and `fix`. The overall level is the worst finding.

Display (`renderCheck`): its own collapsible section `#secCheck`, tinted amber, red or green by the overall level. Collapsed, it lists each finding's title with a ⚠; open, each finding is a white card with a small uppercase label (`WORTH A LOOK · FLOUR STRENGTH`), the title, the explanation and a bold fix that names the control to change. With no findings it shows one green line: "`48 h, W 298 and 66% suit each other.`"

Inputs: `blend` is the share-weighted W; `strongShare` is the total share of flours at W 350 or above.

### Flour window

`W_WINDOW` gives `[hours, minW, maxW]`, interpolated on `tolerant(h)`:

| Hours | Min W | Max W | Basis (knowledge file) |
|---|---|---|---|
| 6 | 220 | 295 | 6.2, and 6.4 (same day: 0–15% strong flour) |
| 12 | 220 | 300 | 6.2 |
| 24 | 260 | 340 | 6.2 |
| 36 | 290 | 340 | 6.4 (75/25 blend, about W 290–305, for 36–48 h) |
| 48 | 290 | 340 | 6.4 and 5.2 |
| 72 | 340 | 400 | 6.2 |

### Rules

| Area | Level | Title | Trigger |
|---|---|---|---|
| Fermentation time | caution | Very long ferment for this flour | h > 51 and blend < 340 |
| Flour, hydration and time | **risk** | Risk of a slack, unworkable dough | 42 ≤ h ≤ 51, hyd ≥ 68, blend < 280 |
| Flour strength | caution, **risk** if 30+ below | Flour too weak for a N h ferment | blend < minW (h ≤ 51) |
| Flour strength | caution | Flour stronger than a N h ferment needs | blend > maxW |
| Hydration | caution | N% is a wet dough to mix by hand | hyd > 68 |
| Fold sets | caution | Wet dough with no fold sets | fold = 0 and hyd ≥ 68 |
| Hydration and flour | caution | Dough may be stiff | hyd ≤ 64 and blend ≥ 340 |
| Flour strength | caution | Too much strong flour for a short ferment | h < 10 and strongShare > 15 |
| Flour strength | caution | Too much very strong flour | strongShare > 30 |
| Kitchen temperature | caution | Warm kitchen: the dough may rise too fast | not cold and temp ≥ 25 |
| Kitchen temperature | caution | Warm kitchen: the final rise may run fast | cold and temp ≥ 27 |
| Pizza style and time | caution | Longer/Shorter than the official Verace rules allow | Verace and (h < 8 or h > 24) |
| Pizza style and flour | caution | Flour outside the official Verace rules | Verace and blend outside 250–320 |

The two Verace-rules findings say explicitly that they're about the AVPN rules, not a dough problem. The **cold fridge** warning is no longer a Dough check rule: picking 2–3 °C shows a small amber note under the Fridge setting, and a ⚠ next to the fridge value in the collapsed summary.

### Strength fixes

`strongFix(target)` suggests the smallest blend step (15%, 25% or 30% of a W 380 flour) that reaches the target **without the total strong share passing 30%**, phrased as "tap Blend… next to Flour strength and add about 25% of a strong W 380 flour" or "raise the stronger flour's share to about N%". If none can, it suggests a shorter ferment or a stronger main flour. Weak-flour warnings on cold schedules add "ball the dough later rather than earlier".

### Sample results

| Dough | Result |
|---|---|
| 48 h, W 298, 66% | Looks good |
| 48 h, W 270, 70%, no folds | Risky: slack dough, wet by hand, no folds |
| 29 h Verace, W 340, 62% | Stiff, plus both Verace-rules notes |
| 8 h, W 310 with 25% strong flour, 27 °C | Stronger than needed, too much strong flour, warm kitchen |

---

## 9. The recipe timeline

`timelineEvents(s, h, r)` builds the steps and `renderTimeline()` groups them by calendar day. Each event has `t` (hours from start), `title`, `pills`, `text` and `exact`.

### Events (cold schedule)

| t | Step | Pills |
|---|---|---|
| 0:00 | Mix | each flour, instant yeast (if weighed), water at the right temperature, solution or fresh yeast |
| 0:20 | Add salt and held-back water, then knead 10 min | salt, held-back water |
| 0:35 | Rest 10 min | |
| 0:45 | Knead again 5–10 min | |
| 1:00 | Into an oiled container | container at 2× the dough volume |
| 1:20, 1:45 | Fold sets (none if folds = 0) | |
| 2:00 | Into the fridge | fridge 4–5 °C |
| h − rise | Ball bowl A | dough balls, oil film or bench flour |
| +0:30 per bowl | Ball bowl B, C… | |
| h − 2.5 | Halfway check | |
| h − 1 | Get ready | semola, mozzarella, toppings, peel |
| h − 0:45 | Poke test, light the oven | |
| h − 0:05 | Measure the stone | stone range (Canotto 430–460 °C, Verace 430–480 °C) |
| h | Open the first pizza (style-specific text), then bake | |
| h + balls × 3 min | Last pizza out | |

The same-day version replaces container, fold and fridge steps with a bulk rise at room temperature.

### Behavior

- **Water temperature:** cold schedules 20–22 °C (18–20 °C at 25 °C+); same-day about 25 °C (20–22 °C at 25 °C+).
- **Bench surface:** oil film at 65% hydration and above, flour below.
- **Uneven bowls** show amounts as `416 / 312 g`.
- **Day groups** are labelled Yesterday, Today or Tomorrow with the date. The first group header has the ±5 min start shifters.
- **Flashing dot:** `markNow()` runs after every render and every 30 s and marks the last started step. Reduced-motion users get a steady ring.

---

## 10. User interface

### Page order

1. **Header:** title centered; a small underlined **Reset** link at top right that swaps to an inline "Reset all? **Yes** No".
2. **Pizza style section** (`#secStyle`)
   - Collapsed: icon list (Pizza style, Hydration, Grams per ball) and a 125 px pizza thumbnail at top right.
   - Open: the **arched selector**, the style photo with an overlaid **?** button (opens a bubble with the style description), Hydration with its hint, Grams per ball.
3. **Fermentation section** (`#secFerm`)
   - Collapsed: "Fermentation: ~29 h, bake Sat 15:00", "Flour strength: W 300, Folds: 2 sets", and the mini graph.
   - Open (titled "Fermentation" with the hours on the right): Start and End, the graph, Flour strength, Fold sets.
4. **Other section** (`#secOther`)
   - Collapsed: one line, "Salt: 2.8% · Fresh 4.8 g · 21° · 4–5°" (fridge left out on same-day plans, ⚠ at 2–3°).
   - Open (titled "Salt" with grams): Salt, Kitchen temperature, Fridge temperature (cold plans only, with the cold-fridge note), Yeast (Fresh/Instant toggle beside the heading, total grams below, "How to test your yeast"), and the weighing method (instant only).
5. **Dough check section** (`#secCheck`), see section 8.
6. **Ingredients:** heading with the ball stepper ("12 balls"); table with Ingredient, per-bowl-size and Total columns and water sub-rows; percentages inline after names; a short note "In grams · 12 × 270 g ≈ 3240 g dough".
7. **Timeline**

Sections are native `<details>`. When open, the section's first control heading becomes the summary title, so there's no duplicate header. Open state is saved in `state.open`.

### Arched style selector

Two SVG ring segments (`.arc-btn`), with labels on curved `textPath`s. Each button is absolutely positioned over the full arch with `pointer-events: none`; only the segment `path` takes clicks, so the ring's middle isn't clickable. The selected half uses the `#arcGrad` gradient.

The markup is generated: a helper script (`arch.py`, kept in the session scratchpad, not the repo) rebuilds it from ring radius, band thickness, text sizes, width percentage and gap. The final parameters were `R 560, thickness 58, title 15 px, subtitle 9.5 px, width 82%`, with text baselines at 58% and 22% of the band. Change the arch by regenerating the paths rather than hand-editing them.

### Choice lists

| Control | Options (sublabel) |
|---|---|
| Style | Canotto (Tall, puffy rim), Verace (Classic Naples) |
| Ball, Canotto | 250 Smaller, 260 Canotto, 270 Tall rim, 290 Large |
| Ball, Verace | 180 Small, 200 Light, 230 Medium, 250 Full |
| Hydration, Canotto | 64 Easy, 65 Sweet spot, 66 Balanced, 68 Sticky, 70 Wet |
| Hydration, Verace | 58 AVPN, 60 Classic, 62 Easy, 65 Airier |
| Salt | 2.5 Mild, 2.7 Medium, 2.8 Default, 2.9 STG, 3.0 AVPN |
| Kitchen temperature | 19° +20%, 21° +7%, 23° −7%, 25° −20%, 27° −30% |
| Fridge | 2–3° (+30% yeast), 4–5° (Target) |
| Folds | None, 2 sets |
| Flour strength | W stepper ±10 (180–420); blend share ±5 (5–50%) |

### Style descriptions (? bubble)

Each explains the style, then why it shapes the hydration and ball-weight choices. Canotto: contemporary Neapolitan, tall airy "dinghy" rim, 28–30 cm, wetter dough from 64–66%, heavier 260–290 g balls. Verace: AVPN rules, thin centre and 1–2 cm rim up to 35 cm, ruota di carro when stretched wide, 58–65% hydration, 180–250 g balls.

### iOS Safari time field fix

```css
.time { -webkit-appearance: none; appearance: none; width: 100%; max-width: 100%; min-width: 0; }
.time::-webkit-date-and-time-value { min-width: 0; margin: 0; padding: 0; text-align: left; line-height: 44px; }
```

Verified earlier in an iPhone 12 Pro simulator.

---

## 11. Visual design

- **Single light theme** on purpose (owner's request), `color-scheme: light`.
- **Body font:** Apple's system font with `tabular-nums`. No web fonts; Atkinson, Bricolage, IBM Plex and Barlow were all rejected.
- **Title:** a serif stack (`"Iowan Old Style", "Palatino Linotype", Palatino, Georgia, serif`), 1.3 rem, with "Napoli" in italic brick red. Non-Apple devices fall back to Palatino or Georgia.
- **Gutter:** 8 px.
- **Panels:** sections on `--panel` (#f7f5f2); the Dough check tints amber/red/green by level; the blend block uses `--flour-tint`.

### Color tokens (`:root`)

| Token | Value | Use |
|---|---|---|
| `--bg`, `--surface` | `#ffffff` | Page and cards |
| `--panel` | `#f7f5f2` | Sections, pills |
| `--tint` / `--tint-line` | `#fff5ee` / `#f4dccb` | Help button, small accents |
| `--fg` / `--muted` / `--line` | `#1f1a17` / `#5f5750` / `#e6e0d8` | Text and lines |
| `--accent` / `--accent-end` | `#a32a12` / `#c2401f` | Numbers, selected choices, arch gradient |
| `--green*` | `#256b43` … | "Use current time", good check |
| `--caution*` / `--warn*` | amber / red | Dough check |
| `--flour-tint` | `rgba(176,115,15,.08)` | Blend block background |
| `--c-*` | per icon | Icon colors (flour amber, yeast purple, water blue, salt slate, pizza tomato, bread, fridge teal, temperature pink, scale green, bubbles sky, flame orange, clock indigo) |

Picker row colors: lunch yellow (`#b07a00` on `#fff7df`), midday orange (`#c4561a` on `#fdeee3`), dinner indigo (`#4b4fc4` on `#eef0ff`). Graph: amber `#c98a2e`/`#dca04a`/`#ecb968`, fridge blue `#5aa7cc`/`#3e86b6`, rise to `#cf4a22`, night navy overlay.

---

## 12. Icons and licenses

Icons are inlined as an SVG sprite and used through the `ico(name)` helper. Each takes its color from its `.i-NAME` class (or `currentColor` where noted).

| Name | Source | License |
|---|---|---|
| `flour` | Game Icons `flour` (sack) | **CC BY 3.0**, attribution required |
| `yeast` | Material Design Icons `yeast` | Apache 2.0 |
| `wheat`, `water`, `salt`, `pizza`, `bread`, `fridge`, `temp`, `scale`, `flame`, `clock`, `alert`, `sun`, `sunset`, `moon` | Tabler | MIT |
| `bubbles` | Lucide | ISC |

The README credits Game Icons but not the individual artist (couldn't be confirmed). The flour sack is also more detailed than the outline icons.

---

## 13. Deployment

- GitHub Pages serves `index.html` and `images/` from the root of `main`. A push typically goes live in 40–50 s.
- Commits end with the `Co-Authored-By: Claude …` trailer. **Push only when the owner says so.**
- `.claude/launch.json` (local preview config for the desktop app's browser pane) is deliberately not committed.

```bash
cd /Users/Manuel/Documents/NapoliDoughCalc
python3 -m http.server 8765 --bind 127.0.0.1   # local preview at http://127.0.0.1:8765/
git push origin main                            # publish
```

---

## 14. Testing approach

No test framework yet.

1. **Syntax check** of the inline script after every edit:
   ```bash
   sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > "$TMPDIR/page.js" && node --check "$TMPDIR/page.js"
   ```
   It has caught real bugs (a duplicate `const`, a duplicated block).
2. **Pure logic in Node:** extract the CALC block, append scenario calls (e.g. `checkDough` on sample doughs) and print the results. Used to proofread every Dough check message.
3. **Real rendering:** a local server and the desktop app's browser pane at 375×812. Add a `?v=N` query to bust the cache after edits.

**Owner's preference:** the owner tests UI changes themselves. Keep browser checks to one quick screenshot unless asked to iterate, and don't leave the pane's saved plan altered.

**Gaps:** no automated regression suite, no real-device test of the new layout, no check of the flashing dot over real time.

---

## 15. Estimates and assumptions to verify

| Item | Current value | Status |
|---|---|---|
| Kitchen temperature factors at 21 and 23 °C | ×1.07 and ×0.93 | Interpolated |
| 27 °C factor | ×0.7 | Matches "subtract about 30%" |
| Yeast between 8 and 24 h | interpolated | No tested schedule there |
| Cold-fridge multiplier | ×1.3 | Top of the file's 25–30% |
| Default flour | W 300 (Canotto), W 270 (Verace) | Owner's choice |
| Ball choices and labels | | Developer's wording |
| Dough check thresholds between table rows | interpolated | |
| Held-back water for small bowls (5%) | | Developer's rule |
| Same-day rise under 6 h (h / 2) | | Developer's rule |
| Picker meal times and labels (Midday = 15–18) | | Owner's choice |

---

## 16. Known issues and gotchas

- **Flour icon credit** is incomplete (section 12).
- **The yeast dose isn't checked**, because the app sets it itself.
- **`endDay` is capped at 2**; ferments over about 72 h can't be entered. Day-after dinner chips can reach ~57 h and trigger the long-ferment warning.
- **Graph labels:** on 48 h plans the fridge-in time is nudged away from its tick; the ☾ can touch the FRIDGE label when night falls mid-fridge.
- **CSS ordering:** several earlier fixes failed because a later general rule (e.g. `.link`, `.head-row`) overrode a new rule of equal specificity. Use a more specific selector rather than relying on order.
- **Editing workflow:** large edits are done with Python string replacement that asserts exactly one match. Regex replacements across the two arch buttons once swapped their shapes; regenerate the arch with the helper instead.
- **The private Claude artifact is stale.**
- **localStorage on iOS** can be cleared after 7 days without a visit.
- **Rounded dough total** in the Ingredients note can be a gram off.

---

## 17. Backlog and ideas

1. **Tick-off timeline:** mark steps done, saved during a two-day ferment
2. **Bake log:** the knowledge file's 18.4 template, to tune yeast over time
3. **"What went wrong?" helper** from section 16 of the knowledge file
4. **Shopping and equipment checklist** (section 18.1)
5. **Redraw the flour icon** as an outline and fix the credit
6. **Automated tests:** commit the Node scenario scripts as a test file
7. **Manual yeast override** with a dose check
8. **Home-screen web app** (manifest and icon) for better storage survival on iOS
9. **Smaller image files:** the 2048 px webp photos are shown at most ~350 px wide
10. **Commit the arch generator** if the selector will be tweaked again

---

## 18. Reference figures

From the knowledge file, for quick checks:

- **Recipe A, 48 h, 12 balls:** 1,440 g Caputo Pizzeria + 480 g Manitoba, 1,267 g water (66%), 54 g salt, 6 g fresh (0.30%) or 2.0 g instant (0.10%). Fridge 40–44 h at 4–5 °C, ball 5–6 h before baking, stone 430–460 °C.
- **Recipe B, 24 h, 3 balls:** 410 g Caputo + 70 g Manitoba (about 15%), 317 g water, 13 g salt, 1.0–1.2 g fresh. 2 h warm, about 18 h fridge, ball 5 h before.
- **Recipe C, same day, 3 balls:** 490 g Caputo, 318 g water (65%), 14 g salt, 2 g fresh (0.4%). Bulk until 2:30, then about 4.5 h appretto.
- **Rules:** STG 180–250 g balls, 4–6 h appretto, 60–90 s bake. AVPN 2022: 8–24 h total, 40–60 g salt per litre, 0.1–3 g fresh yeast per litre. Canotto: 65–78% hydration, about 260–270 g, up to 31 cm.
- **The non-negotiables (file 17.7):** test unsure yeast, weigh yeast directly, measure the fridge, knead about 15–20 min in total, ball 4–6 h before, proof until doubled, keep the stone at 430–460 °C, and never press the rim.
