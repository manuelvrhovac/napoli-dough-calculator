# Napoli Pizza Calculator: technical writeup

This document records everything built so far: what the app does, how every calculation works, where each number comes from, how the page is structured, how it's deployed and tested, and what is still open. It's written so a new working session can continue without the original conversation.

- **Live site:** https://manuelvrhovac.github.io/napoli-dough-calculator/
- **Repository:** https://github.com/manuelvrhovac/napoli-dough-calculator (branch `main`, published by GitHub Pages from `main`)
- **Local checkout:** `/Users/Manuel/Documents/NapoliDoughCalc`
- **Main file:** `index.html` (about 950 lines, single file, no build step, no dependencies)
- **Source documents:** `docs/sources/` (see [Source documents](#2-source-documents))
- **Last pushed commit when this was written:** `36f738b Redesign configurator, timeline and theme`

---

## Contents

1. [What the app is](#1-what-the-app-is)
2. [Source documents](#2-source-documents)
3. [How the project evolved](#3-how-the-project-evolved)
4. [Architecture](#4-architecture)
5. [State model and persistence](#5-state-model-and-persistence)
6. [The calculation engine](#6-the-calculation-engine)
7. [Fermentation scheduling](#7-fermentation-scheduling)
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
- **Flours:** Caputo Pizzeria (W 260–280) and K Plus Manitoba (W ~380), with Caputo Nuvola kept for special occasions
- **Yeast:** fresh yeast or Dr. Oetker instant dry (7 g sachets)
- **Typical batches:** 12 balls for a party of 10, or 3 balls for a small dinner

You choose a style, ball count and size, start and bake times, flour mix and formula. The app gives you:

1. A collapsed summary of the key choices
2. A **Dough check** that flags risky combinations of time, flour strength and hydration
3. An **Ingredients** table, split into bowls of up to 6 balls
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

The app was built in about 15 rounds of feedback. Knowing the order helps explain some code shapes.

| # | Change |
|---|---|
| 1 | Basic calculator from the canotto document, published as a private Claude artifact. |
| 2 | Single column for mobile, with a beige, orange and pastel red palette and gradients. |
| 3 | **Auto** fermentation from start and bake times, and hydration as four segmented choices. |
| 4 | Preset choices everywhere: ball weights, five kitchen temperatures, salt choices, and a flour mixer with name, W slider and share. |
| 5 | Flour pick list (placeholders for 20), dated timeline, steps, a **Now** button rounding up to the next 15 min, a green secondary color, and the rename to "Napoli Pizza Calculator". |
| 6 | First Dough check, a rule-based warning system marked work in progress. |
| 7 | **Rebuilt on the knowledge file:** new yeast curve, Canotto and Verace styles, an 8 h same-day option, fridge temperature, yeast weighing method, two-stage kneading and updated check rules. |
| 8 | 22–51 h tolerance, side-by-side Start and End, a fold-set choice, steps merged into the timeline with ingredient pills, a flashing current-step dot and tighter spacing. |
| 9 | Moved into a git repo, given a standalone HTML head, a fun README, and published on GitHub Pages. |
| 10 | Dough check audited against the knowledge file and rewritten. |
| 11 | Big UX redesign: collapsible configurator with a summary button, Start and End always as input with the hours as suggestions, fresh yeast as the default, an expandable yeast test, a confirmed Reset, groups, a cleaner ingredients table and an iOS time-field fix. |
| 12 | Icons, Apple's system font, ±5 min start shifters and the timeline chain moved left. |
| 13 | Pure white theme without dark mode, colored icons, a darker red for contrast, and the time inline with the step title. |

The private Claude artifact (`https://claude.ai/artifact/Mgn9k5pry6UYXVMF3D4BB9`) stopped being updated after round 8. **The repository is the only up-to-date copy.**

---

## 4. Architecture

One HTML file with inline CSS and JS. There's no framework, bundler or network dependency at runtime.

```
index.html
├── <head>: charset, viewport (viewport-fit=cover), base reset, page <style>
├── <svg> sprite: 13 <symbol id="i-…"> icons
├── <main class="page">
│   ├── header (centered title)
│   ├── section#config (collapsible configurator)
│   ├── section#check (Dough check)
│   ├── section Ingredients (table + note)
│   └── section Timeline
└── <script>
    ├── /* CALC-START */ … /* CALC-END */   pure logic, no DOM
    ├── data: FLOUR_LIST, STYLES, HYD_WHY
    ├── state load, migration and validation
    ├── date helpers, timeline builder, renderers
    └── event delegation (click, input, change)
```

### The pure-logic block

Everything between `/* CALC-START */` and `/* CALC-END */` is pure: no DOM, no globals outside the block. This is deliberate, so it can be pulled out with `sed` and tested in Node:

```bash
sed -n '/CALC-START/,/CALC-END/p' index.html > /tmp/calc.js
```

It contains: `interp`, `roundTo`, `FRESH_CURVE`, `tolerant`, `schedule`, `toMin`, `autoHours`, `TEMP_FACTOR`, `MAX_PER_BOWL`, `defaultFlours`, `DEFAULTS`, `shares`, `blendW`, `weigh`, `calculate`, `W_WINDOW`, `BLEND_STEPS` and `checkDough`.

### Rendering model

- `render()` recomputes everything from `state` and writes the DOM. It's called after every change.
- `renderFlours()` rebuilds the flour list only on structural changes (add, remove, expand, share, pick), so dragging the W slider or typing a name doesn't lose focus. `refreshFlours()` calls both.
- `renderStyle()` rebuilds the ball and hydration choices, which depend on the style.
- All interaction uses **one delegated `click` listener**, plus `input` (W slider, flour name) and `change` (flour pick list, time inputs).
- Segmented choices are `<div class="seg" id="KEY">` containing `<button data-v="VALUE">`. The generic handler writes `state[KEY] = value`, parsed as a number when numeric. The selected state is `aria-pressed="true"`, set in `render()`.
- `[hidden]` is forced to `display: none !important` by the base reset, so visibility is toggled with `el.hidden`.

---

## 5. State model and persistence

```js
DEFAULTS = {
  style: "canotto", balls: 12, ball: 270, hyd: 66, salt: 2.8, temp: 21, fridge: 4,
  yeast: "fresh", weigh: "scale", fold: 2,
  startTime: "17:00", endDay: 2, endTime: "17:00"
}
// plus: startDate ("YYYY-MM-DD", today), flours (array), cfgOpen (bool, default false)
```

| Field | Meaning |
|---|---|
| `style` | `"canotto"` or `"verace"`. It changes the ball and hydration choices, the opening text and the stone range. |
| `balls` | 1–36 (stepper) |
| `ball` | grams per ball, from the style's list |
| `hyd`, `salt` | percentages of flour |
| `temp` | kitchen °C: 19, 21, 23, 25 or 27 |
| `fridge` | `3` (meaning 2–3 °C) or `4` (meaning 4–5 °C) |
| `yeast` | `"fresh"` or `"dry"` (instant) |
| `weigh` | `"scale"` or `"solution"` (dry yeast only) |
| `fold` | 0, 2 or 3 fold sets |
| `startDate`, `startTime` | when mixing starts |
| `endDay`, `endTime` | bake day, as 0, 1 or 2 days after `startDate`, and the time |
| `flours` | `[{ name, w }, { name, w, share }, …]`. The first flour has no share and takes the rest. |
| `cfgOpen` | whether the configurator is expanded |

**Fermentation hours are derived, never stored:** `autoHours(s) = (endDay·1440 + endTime − startTime) / 60`.

### Persistence

- Key `canotto-calc-v1` in `localStorage`, written at the end of every `render()` and wrapped in try/catch.
- **The start date is stored**, so a saved plan keeps its real dates. If `startDate` isn't today, a hint shows the saved start.
- Data is per browser and per device. On iOS Safari, script-written storage can be cleared after 7 days without a visit, and Private Browsing keeps nothing.
- The origin `manuelvrhovac.github.io` is shared with any other Pages project on that account. The key name keeps it separate.

### Migration and validation on load

Old saved shapes are repaired, so users never see a broken page:

- `ball`, `hyd`, `salt` and `temp` snap to the **nearest** allowed value for the current style (`nearest()`). Old strings such as `temp: "normal"` fall back to the default.
- An old fixed `hours` (8, 24, 36 or 48) is converted to an end time with `setEnd()`, then deleted. `hours: "auto"` is simply deleted.
- An invalid `startDate`, `fold`, `style` or `flours` is reset to the default.
- `cfgOpen` defaults to `false`.

Users who saved `yeast: "dry"` before fresh became the default keep dry until they change it or tap Reset.

---

## 6. The calculation engine

### 6.1 Scaling (dough factor)

From the knowledge file (section 3.2):

```
total dough  = balls × ball weight
dough factor = 1 + hyd/100 + salt/100 + yeast%/100      (instant % if dry, fresh % if fresh)
flour        = total dough ÷ dough factor
```

Implementation details:

- **Bowls:** `bowlCount = ceil(balls / 6)`. Balls are split as evenly as possible into at most two bowl sizes, `big` and `big − 1`. For example, 7 balls gives one bowl of 4 and one of 3, and 12 gives two of 6.
- **Flour per bowl** is rounded to the nearest **10 g**: `roundTo(bowlBalls × ball / factor, 10)`. Total flour is the sum of the bowls.
- **Per bowl, through `weigh()`:**
  - Added flours: `round(flour × share%)`. The main flour is the remainder, so the parts always add up exactly.
  - Water `round(flour × hyd%)`, salt `round(flour × salt%)`.
  - Yeast: `flour × fresh% / 100`, divided by 3 for dry yeast (the AVPN 1:3 ratio).
  - **Held-back water:** `clamp(roundTo(water × 5%, 5), 5, 30)`. That's 30 g per bowl for normal batches, from the document's bassinage step, and less for small bowls.
  - **Solution** (dry yeast with the solution method): a 7 g sachet made up to 70 g with water, so 1 g of solution carries 0.1 g of yeast. The solution amount is `max(0.5, roundTo(yeast × 10, 0.5))` grams.
  - **Plain water in the bowl** = water − solution − held-back water.
- **Totals:** water and salt are recomputed from total flour, which matches the documents' totals (1,267 g rather than 2 × 634). Solution and held-back water are summed. Plain water is the total water minus those.

**Check:** 12 × 270 g at 66% and 2.8% gives 1,920 g flour (1,440 + 480), 1,267 g water and 54 g salt, matching both documents' tables. 3 balls gives 480 / 317 / 13.

### 6.2 Yeast dose

**Fresh-equivalent % of flour, by fermentation hours** (`FRESH_CURVE`, from knowledge file section 17.6), linearly interpolated and clamped at the ends:

| Hours | Fresh % | Instant % (÷3) | Source |
|---|---|---|---|
| ≤ 8 | 0.40 | 0.133 | Recipe C, same day |
| 24 | 0.24 | 0.08 | 24 h row (0.20–0.25) |
| 36 | 0.27 | 0.09 | 36 h row (0.25–0.30) |
| ≥ 48 | 0.30 | 0.10 | 48 h row and Recipe A |

Longer cold ferments use slightly **more** yeast than 24 h, as the knowledge file explains: fridge hours count for little, so the yeast does most of its work in the warm windows at each end.

**Multipliers:**

- **Kitchen temperature** (`TEMP_FACTOR`): 19 °C ×1.2, 21 °C ×1.07, 23 °C ×0.93, 25 °C ×0.8 and 27 °C ×0.7. See [estimates](#15-estimates-and-assumptions-to-verify) for where these come from.
- **Cold fridge** (2–3 °C) on cold schedules: ×1.3, the top of the file's "add about 25–30%".

**Tolerance:** `tolerant(h)` maps 22–24 h to 24 h and 48–51 h to 48 h for yeast and flour windows, so small overshoots don't change anything.

### 6.3 Display rounding

- Instant yeast under 3 g shows 2 decimals (for a 0.01 g scale). Other amounts under 10 g show 1 decimal, and larger amounts are whole grams.
- Percentages next to yeast show the effective dose, such as `0.32%` fresh at 21 °C for 48 h.

---

## 7. Fermentation scheduling

`schedule(h)` returns `{ cold, fridgeAt, ballAt, rise, fresh }`.

| Hours | Type | Plan |
|---|---|---|
| under 10 | Room temperature only | Bulk until `ballAt`, then a final rise. From 6 h, `rise = h − 2.5` (bulk ends at 2.5 h). Under 6 h, `rise = h / 2`. |
| 10 and up | Cold bulk | 2 h warm, fridge, ball at `h − rise`, where `rise` is 5 h at 24 h, 5.5 h at 36 h and later, interpolated and rounded to 0.25 h. |

- The **minimum** is 4 h. Under that, the timeline and check show a prompt instead.
- **9–21 h** works, but the Dough check marks it as between the main schedules, because the yeast and timing there are interpolated.
- **Over 51 h** works, but gets a warning that it needs W 340+ flour and that yeast stops at the 48 h amount.

### Start and End

- Start and End are always the inputs. The 8, 24, 36 and 48 h buttons are **suggestions** that set End through `setEnd(hours)`. A suggestion is highlighted only when the current hours equal it exactly.
- **Use current time** sets the start to now, rounded **up** to the next 15 min (`nextQuarter`). For example, 18:19 and 18:23 both give 18:30, and 18:15 stays.
- **±5 min shifters** (`shiftStart`) move only the start, and the bake time stays where it is. If the start crosses midnight, `endDay` is adjusted so the bake stays at the same absolute time.
- Changing the start time by hand resets `startDate` to today.

---

## 8. The Dough check

`checkDough({ hours, blend, hyd, strongShare, style, fold, temp, fridge })` returns `{ level, findings[], minW, maxW }`. Each finding has `level`, `title`, `why` and `fix`. The overall level is the worst finding, shown as **Looks good**, **Worth a look** or **Risky**. It's marked work in progress in the UI.

Inputs: the blend W is the share-weighted average of the flours, and `strongShare` is the total share of flours at W 350 or above.

### Flour window

`W_WINDOW` gives `[hours, minW, maxW]`, interpolated on `tolerant(h)`:

| Hours | Min W | Max W | Basis (knowledge file) |
|---|---|---|---|
| 6 | 220 | 295 | 6.2, and section 6.4 (same day: 0–15% Manitoba) |
| 12 | 220 | 300 | 6.2 |
| 24 | 260 | 340 | 6.2 (260–300 for 12–24 h, 300–340 for 24–48 h) |
| 36 | 290 | 340 | 6.4 (75/25 blend, about W 290–305, for 36–48 h) |
| 48 | 290 | 340 | 6.4 and 5.2 (48 h recipes use W 280+) |
| 72 | 340 | 400 | 6.2 (W 340–400 for 48–72 h) |

### Rules

| Rule | Level | Trigger |
|---|---|---|
| Past 51 h needs very strong flour | caution | h > 51 and blend < 340 |
| Between the main schedules | caution | 8.5 < h < 22 |
| Long, wet and too weak | **risk** | 42 ≤ h ≤ 51, hyd ≥ 68 and blend < 280 |
| Flour is on the weak side | caution, or **risk** if 30+ below | blend < minW (h ≤ 51) |
| Flour is stronger than needed | caution | blend > maxW |
| Wet dough by hand | caution | hyd > 68 |
| Wet dough without folds | caution | fold = 0 and hyd ≥ 68 |
| Dough may be stiff | caution | hyd ≤ 64 and blend ≥ 340 |
| Too much strong flour (same day) | caution | h < 10 and strongShare > 15 |
| A lot of very strong flour | caution | strongShare > 30 |
| Warm kitchen for same-day dough | caution | not cold and temp ≥ 25 |
| Final rise may run fast | caution | cold and temp ≥ 27 |
| Cold fridge | caution | cold and fridge = 3 |
| Outside the AVPN time (Verace) | caution | h < 8 or h > 24 |
| Outside the AVPN flour range (Verace) | caution | blend < 250 or > 320 |

### Strength fixes

`strongFix(target)` suggests the smallest of the file's blend steps (15%, 25% or 30% of a W 380 flour) that reaches the target **without the total strong share passing 30%**. If none can, it suggests a shorter ferment (the longest window the blend fits) or a stronger main flour. Weak-flour warnings on cold schedules add "ball it later rather than earlier" (knowledge file 12.1).

### Design decisions from the audit

An earlier version had a made-up hydration formula (`65% + 1% per W 20`) and marked 70% on W 298 as Risky. That contradicted the file, where Vitiello uses 70% on W 280+ and Pizza Mon Amour uses 71%. It also suggested "raise the blend to W 350", which breaks the 30% Manitoba cap. **Hydration is now judged by the file's ladder** (up to 68% is fine, and above that gets a handling caution), and Risky is reserved for combinations the file actually warns about.

### Sample results

| Dough | Result |
|---|---|
| 48 h, W 298, 66% (party dough) | Looks good |
| 48 h, W 298, 68% | Looks good |
| 24 h, W 298, 70% | Worth a look (wet by hand) |
| 36 or 48 h, 100% Caputo Pizzeria | Worth a look: use 25% W 380 or shorten to 24 h |
| 48 h, 100% Caputo Pizzeria, 70% | Risky |
| 8 h with 25% Manitoba | Worth a look: too strong for same day |
| 24 h, 100% Manitoba, 64% | Worth a look: too strong, stiff, over 30% strong |

---

## 9. The recipe timeline

`timelineEvents(s, h, r)` builds the list of steps. `renderTimeline()` groups them by calendar day and renders them. Each event has `t` (hours from start), `title`, `pills` (ingredients with amounts), `text` and `exact` (round to the minute, not 5 min).

### Events (cold schedule)

| t | Step | Pills |
|---|---|---|
| 0:00 | Mix | each flour, instant yeast (if weighed), water at the right temperature, solution or fresh yeast |
| 0:20 | Add salt and held-back water, then knead 10 min | salt, held-back water |
| 0:35 | Rest 10 min | |
| 0:45 | Knead again 5–10 min | |
| 1:00 | Into an oiled container | container at 2× the dough volume |
| 1:20, 1:45 | Fold sets (with 3 sets: 1:15, 1:35, 1:55) | |
| 2:00 | Into the fridge | fridge 4–5 °C |
| h − rise | Ball bowl A, with what to expect after cold bulk | dough balls, oil film or bench flour |
| +0:30 per bowl | Ball bowl B, C… (staggered to match baking order) | |
| h − 2.5 | Halfway check | |
| h − 1 | Get ready | semola, mozzarella, toppings, peel |
| h − 0:45 | Poke test, light the oven (45 min preheat) | |
| h − 0:05 | Measure the stone | stone range |
| h | Open the first pizza, then bake | |
| h + balls × 3 min | Last pizza out | |

The same-day version replaces the container, fold and fridge steps with a bulk rise at room temperature, and its folds stop before balling.

### Behavior

- **Water temperature:** cold schedules use 20–22 °C (18–20 °C at 25 °C+). Same-day uses about 25 °C (20–22 °C at 25 °C+).
- **Bench surface:** oil film at 65% hydration and above, flour below, following Esposito's rule in the file.
- **With no folds,** the balling text says to ball tighter.
- **Uneven bowls** show amounts as `416 / 312 g`, and a note above says amounts are per bowl.
- **Day groups** are labelled Yesterday, Today or Tomorrow with the date, or just the date.
- **Flashing dot:** each `<li>` has `data-at` (epoch ms). `markNow()` runs after every render and every 30 s, and adds `.live` to the last started step's dot. It's off before the first step and after the last one. Reduced-motion users get a steady ring.

---

## 10. User interface

### Page order

1. **Title**, centered
2. **Configurator** (`section#config`)
   - **Collapsed:** one summary button (`#cfgOpen`) with six tiles (dough balls, fermentation, flour W, hydration, style, bake day and time) and a faint chevron as the only expand hint
   - **Expanded:** a Collapse button at the top, then five groups on faint panels:
     1. Pizza style, Dough balls (stepper), Grams per ball
     2. Fermentation: Start and End side by side, a flag line, Suggested hours and a plan hint
     3. Flour mix: expandable flour cards with a pick list, name, full-width W slider (180–420) and share (10–30%), plus Add a flour (up to 4)
     4. Hydration, Salt, Kitchen temperature, Fridge temperature (cold schedules only), and Yeast (Fresh, Instant dry) with a "How to test your yeast" `<details>`
     5. Other: how you weigh the yeast (dry only) and Fold sets
   - **Footer:** Collapse and a red **Reset** that asks for confirmation inline ("Reset all choices?" with Cancel and Yes, reset)
3. **Dough check**
4. **Ingredients:** table with an Ingredient column, a per-bowl-size column and a highlighted Total column, with water sub-rows. A note below gives the total dough weight.
5. **Timeline**

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
| Suggested hours | 8 Same day, 24 Most reliable, 36 In between, 48 Most flavour |
| Folds | None, 2 sets (Recommended), 3 sets |
| Flour share | 10, 15, 20, 25, 30% (options that would leave the main flour at 0% are disabled) |

Switching style resets hydration and ball weight to that style's defaults: Canotto 66% and 270 g, Verace 60% and 250 g.

### Flour pick list

`FLOUR_LIST` has 20 slots. Nine are filled, and **11 are `null` placeholders** waiting for research, shown as one disabled "11 more flours to be added" option.

| Flour | W used | Note |
|---|---|---|
| Caputo Pizzeria | 270 | Mill: W 260–280 |
| Caputo Nuvola | 270 | Mill: W 260–280 |
| Caputo Nuvola Super | 320 | File: W 300–340 class |
| Caputo Cuoco | 320 | File: W 300–340 class |
| Caputo Manitoba Oro | 370 | Unverified |
| K Plus Manitoba | 380 | File |
| Le 5 Stagioni Pizza Napoletana | 270 | Unverified |
| Le 5 Stagioni Superiore | 300 | Unverified |
| Polselli Classica | 300 | Unverified |

### iOS Safari time field fix

`<input type="time">` overflowed its column on iOS Safari. The fix:

```css
.time { -webkit-appearance: none; appearance: none; width: 100%; max-width: 100%; min-width: 0; }
.time::-webkit-date-and-time-value { min-width: 0; margin: 0; padding: 0; text-align: left; line-height: 44px; }
```

This was verified in an iPhone 12 Pro simulator. The `line-height` re-centers the value vertically, because removing the native appearance top-aligns it.

---

## 11. Visual design

- **Single light theme** on purpose, with pure white background and no dark mode, as the owner requested. `color-scheme: light` is set in the base reset.
- **Font:** Apple's system font (`-apple-system, BlinkMacSystemFont, "SF Pro Text"/"SF Pro Display", "Helvetica Neue", system-ui`) with `tabular-nums`. No web fonts are loaded. The project went through Atkinson, Bricolage and IBM Plex, then Barlow, and the owner rejected both as too wide or unappealing.
- **Gutter:** 8 px. The owner asked for less padding than the usual 16 px.

### Color tokens (`:root`)

| Token | Value | Use |
|---|---|---|
| `--bg`, `--surface` | `#ffffff` | Page and cards |
| `--panel` | `#f7f5f2` | Configurator groups, Dough check, pills |
| `--tint` / `--tint-line` | `#fff5ee` / `#f4dccb` | Summary button and day pills |
| `--head` | `#f3efe9` | Table header |
| `--fg` / `--muted` / `--line` | `#1f1a17` / `#5f5750` / `#e6e0d8` | Text and lines |
| `--accent` / `--accent-end` | `#a32a12` / `#c2401f` | Dark brick red for numbers and times, and the selected-choice gradient |
| `--green*` | `#256b43` … | "Use current time" and "Looks good" |
| `--caution*` / `--warn*` | amber / red | Dough check findings |
| Reset button | `#c8372b` | Solid red |
| `--c-*` | per icon | Icon colors: flour amber `#b0730f`, yeast purple `#6f4fc4`, water blue `#1b78c9`, salt slate `#4f6b82`, pizza tomato `#d0401f`, bread `#a9651c`, fridge teal `#0d8ca6`, temperature pink `#d12f6c`, scale green `#0f8a6c`, bubbles sky `#2f9bd6`, flame orange `#e8590c`, clock indigo `#4b57d1` |

---

## 12. Icons and licenses

Icons are inlined as an SVG sprite at the top of `<body>`. They're fetched from the Iconify API and stored as `<symbol id="i-NAME">`, then used with `<svg class="ico i-NAME"><use href="#i-NAME"/></svg>`, through the `ico(name)` helper in JS. Each one takes its color from the matching `.i-NAME` class.

| Name | Source | License |
|---|---|---|
| `flour` | Game Icons `flour` (sack) | **CC BY 3.0**, attribution required |
| `yeast` | Material Design Icons `yeast` | Apache 2.0 |
| `wheat`, `water` (droplet), `salt`, `pizza`, `bread`, `fridge`, `temp` (temperature), `scale`, `flame`, `clock` | Tabler | MIT |
| `bubbles` | Lucide | ISC |

The README has a Credits section. **It credits Game Icons but not the individual artist**, because the artist couldn't be confirmed (game-icons.net didn't respond). The flour sack is also more detailed and solid than the outline icons. Replacing it with a simple custom outline would fix both.

---

## 13. Deployment

- GitHub Pages serves `index.html` from the root of `main`. A push typically goes live in 40–50 s.
- The standalone page needed its own `<!doctype html>`, charset and viewport meta, and a small reset (`[hidden]`, body margin, safe-area padding). The Claude artifact host had been adding those automatically.
- Commits end with the `Co-Authored-By: Claude …` trailer. Pushing happens only when the owner says so.

Useful commands:

```bash
cd /Users/Manuel/Documents/NapoliDoughCalc
python3 -m http.server 8765 --bind 127.0.0.1   # local preview at http://127.0.0.1:8765/
git push origin main                            # publish (Pages rebuilds automatically)
```

---

## 14. Testing approach

No test framework yet. Three layers have been used.

1. **Syntax check** of the whole inline script:
   ```bash
   sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > /tmp/page.js && node --check /tmp/page.js
   ```
   This caught a real bug once, a duplicate `const` that broke a published version.
2. **Pure logic tests:** extract the CALC block and run scenarios in Node (scaling tables, `schedule()`, the yeast curve, `checkDough()` against knowledge-file cases).
3. **Page script in a stub DOM:** run the full script in `vm.runInNewContext` with a fake `document.getElementById` and `localStorage` (including old saved shapes), then read `textContent` and `innerHTML` of the timeline, table and summary.
4. **Real rendering:** a local server, the desktop app's browser pane at 375×812 (click flows checked with JS: suggestions, ±5, reset with confirmation, collapse, horizontal overflow), and the iOS Simulator (iPhone 12 Pro) for Safari-specific layout.

**Gaps:** no automated regression suite, no real-device test, and no check of the flashing dot over real time.

---

## 15. Estimates and assumptions to verify

These numbers aren't directly from the sources and should be checked against real bakes:

| Item | Current value | Status |
|---|---|---|
| Kitchen temperature factors at 21 and 23 °C | ×1.07 and ×0.93 | Interpolated between the file's +20% at 19 °C and −20% at 25 °C |
| 27 °C factor | ×0.7 | Matches the file's "summer 26 °C+: subtract about 30%" |
| 22 °C baseline | not offered | The owner asked for 19/21/23/25/27, so the default 21 °C adds 7% |
| Yeast between 8 and 24 h | interpolated | The file has no tested schedule there |
| Cold-fridge multiplier | ×1.3 | Top of the file's 25–30% |
| Ball choices and labels (Verace sizes, 250 and 290 canotto) | | Labels are the developer's wording |
| W values for 7 of 9 flours | | Unresearched |
| Dough check thresholds between table rows | interpolated | |
| Held-back water for small bowls (5% of water) | | Developer's rule; the documents only give 30 g per bowl |
| Same-day rise when under 6 h (h / 2) | | Developer's rule |

---

## 16. Known issues and gotchas

- **Flour icon credit** is incomplete (see [section 12](#12-icons-and-licenses)).
- **Caputo Nuvola** is judged by W only, so it gets weak-flour warnings at 36–48 h, even though it's sold for 24 h or more.
- **The yeast dose isn't checked**, because the app sets it itself. A manual override would need a check.
- **Same-day dough over 27 °C** could over-proof. Only a caution covers it.
- **`endDay` is capped at 2** (Day after). Ferments over about 72 h can't be entered.
- **The `index.html` editing workflow:** large edits were done with Python string replacement that asserts exactly one match. One slice used `t.index()` without a start offset and duplicated a block; it was caught by `node --check`. Prefer exact unique anchors and always run the syntax check.
- **The private Claude artifact is stale.** Don't update it as if it were current.
- **localStorage on iOS** can be cleared after 7 days without a visit.
- **Mixed icon styles:** solid yeast and flour icons next to Tabler outlines.

---

## 17. Backlog and ideas

Offered to the owner but not built yet:

1. **Tick-off timeline:** mark steps done, saved during a two-day ferment
2. **Bake log:** the knowledge file's 18.4 template, so yeast can be tuned over time
3. **"What went wrong?" helper:** symptom to cause to fix, from section 16 of the file
4. **Shopping and equipment checklist** (section 18.1)
5. **Research and fill the 11 flour slots**, and verify the W values
6. **Redraw the flour (and possibly yeast) icon** as outlines to match Tabler, and fix the credit
7. **Optional dark theme** again, if wanted later
8. **Automated tests:** turn the Node scenario scripts into a committed test file
9. **Nuvola-aware flour check**, and a manual yeast override with a dose check
10. **Home-screen web app** (manifest and icon) for better storage survival on iOS

---

## 18. Reference figures

From the knowledge file, for quick checks:

- **Recipe A, 48 h, 12 balls:** 1,440 g Caputo Pizzeria + 480 g Manitoba, 1,267 g water (66%), 54 g salt, 6 g fresh (0.30%) or 2.0 g instant (0.10%). Fridge 40–44 h at 4–5 °C, ball 5–6 h before baking, stone 430–460 °C.
- **Recipe B, 24 h, 3 balls:** 410 g Caputo + 70 g Manitoba (about 15%), 317 g water, 13 g salt, 1.0–1.2 g fresh. 2 h warm, about 18 h fridge, ball 5 h before.
- **Recipe C, same day, 3 balls:** 490 g Caputo, 318 g water (65%), 14 g salt, 2 g fresh (0.4%). Bulk until 2:30, then about 4.5 h appretto.
- **Rules:** STG 180–250 g balls, 4–6 h appretto, 60–90 s bake. AVPN 2022: 8–24 h total, 40–60 g salt per litre, 0.1–3 g fresh yeast per litre. Canotto: 65–78% hydration, about 260–270 g, up to 31 cm.
- **The non-negotiables (file 17.7):** test unsure yeast, weigh yeast directly, measure the fridge, knead about 15–20 min in total, ball 4–6 h before, proof until doubled, keep the stone at 430–460 °C, and never press the rim.
