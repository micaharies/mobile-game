# Prediction markets spec

Companion to `DESIGN_NOTES.md`. Fleshes out the prediction-market asset
type into a real system with variety and structure, instead of the two
hardcoded example markets currently in `ASSET_DEFS`. Covers content
categories, resolution mechanics, pricing behavior, and how this ties
into the existing tiered-unlock and market-event systems.

## Why predictions specifically

Per `DESIGN_NOTES.md`, prediction markets were chosen as a core pillar
because they're fast-resolving, easy to generate absurd content for, and
fit mobile session length better than slow-moving real markets. Right
now that potential is basically unused — two static markets isn't enough
to carry the tone or the pacing. This spec turns "prediction markets" into
an actual content system, not just an asset type.

## Market categories

Split into distinct flavors so the list doesn't read as one repetitive
bucket. Each category has its own tone and its own typical resolution
window.

### 1. Absurd/satirical (the game's signature content)
Fictional, deliberately ridiculous, matches the game's satirical voice
established in `OPENING_SEQUENCE_SPEC.md`. This is the category that
should make up the bulk of the list and do the most work
tonally.
- "Will the office microwave survive another popcorn incident?"
- "Will Gary the pit boss get promoted?" (callback to the opening
  sequence's recurring narrator-adjacent character — reuse recurring
  fictional characters here for continuity, see Notes below)
- "Will a raccoon get back into the trash cans this week?" (callback to
  the community-service minigame's raccoon flavor)
- Resolution window: short, minutes to a few hours — these are meant to
  be quick, high-frequency, low-stakes fun

### 2. Fictional "current events" (in-universe, not real)
Invented news-style events that feel current without referencing real
people, places, or brands (keeping the copyright/satire rules from
`OPENING_SEQUENCE_SPEC.md` — "don't reference real casinos, real people,
or real brands" — applies here too).
- "Will the city council approve the new stadium?"
- "Will the mystery lawsuit against [fictional megacorp] settle this
  week?"
- Resolution window: medium, several hours to a day

### 3. Weather/environment (procedurally generated, not fictional)
These can actually be real and locally relevant if you want a
"live-feeling" category — or kept fully simulated for simplicity. Either
works; pick one for v1 rather than building both:
- **Simulated (recommended for v1):** server/client generates a random
  weather-style outcome with a plausible-feeling probability, no real
  data dependency — simplest, no API cost, no rate limits, fits the
  "simulated market" philosophy the whole game already uses for price
  ticks.
- **Real (later upgrade):** pull from a free weather API for the
  player's region/a fixed reference city; adds actual live-data flavor
  but introduces an external dependency this game hasn't needed
  anywhere else yet. Not recommended for the POC/alpha stage — flag as
  a v2 idea only.
- Resolution window: short, hours (aligns with realistic weather
  timeframes)

### 4. Meta/self-referential (about the game itself)
Predictions about the player's own in-game behavior or the market
state — this is a fun category specific to this game since it can
reference real, live state rather than being purely scripted.
- "Will the player go broke before midnight?" (references actual cash
  trajectory)
- "Will DOGO close green today?" (references an actual tracked asset's
  actual price, resolved against real simulated data — this is the one
  category where resolution should hook into real game state rather
  than being scripted/random)
- Resolution window: variable, tied to whatever it's referencing

### 5. Short-fuse "hype" markets (the pacing fix from RETENTION_SPEC.md)
Explicitly the short-fuse rotating market called for in
`RETENTION_SPEC.md` section 2 ("Short-fuse prediction markets") — 60-90
second resolution, one active at a time, rotates to a new one on
resolution. Content pulled from the same pools above (any category can
serve as a hype market), the defining trait is just the short window,
not a separate content category.

## Content pool structure

Rather than hardcoding a fixed list of markets, structure content as a
pool the market-list-generator draws from, so it can rotate and stay
fresh across sessions without manual curation forever.

```js
const PREDICTION_POOL = [
  {
    id: 'microwave-popcorn',
    category: 'absurd',
    prompt: 'Will the office microwave survive another popcorn incident?',
    resolutionWindow: [15, 90],   // minutes, min/max — actual duration randomized within this range
    baseOdds: 0.5,                // starting price/probability before drift
    volatility: 0.035,
  },
  // ...
];
```

- Aim for at least 25-30 entries at launch across the four scripted
  categories (absurd, fictional current-events, weather-simulated,
  meta) so repeat sessions don't feel like the same five markets on
  loop. This is a writing task more than an engineering one — budget
  time for actually drafting the pool, not just the system that
  consumes it.
- At any given time, the markets screen shows a rotating subset (e.g.
  6-10 active at once) rather than the full pool, refreshing as markets
  resolve — keeps the list from feeling like a wall of everything at
  once.
- Meta category prompts need a `resolveAgainst` field referencing real
  game state (e.g. `{ type: 'assetPriceDirection', assetId: 'doge' }`)
  since these resolve against actual simulated data, not a random roll.
  Everything else resolves via weighted random roll against `baseOdds`
  (see Resolution mechanics below).

## Resolution mechanics

**For scripted/random categories (absurd, fictional events, simulated
weather):**
- On market creation, roll a true outcome using `baseOdds` as the
  probability (don't reveal this to the player — it's the "hidden"
  resolution the price should drift toward being predictive of, not
  something exposed).
- Price should drift somewhat toward the true outcome as resolution
  approaches, not be static the whole window — creates the sensation of
  "the market knows something," matching real prediction market
  behavior, without needing to reveal the actual roll early. A simple
  approach: bias the random walk's drift term increasingly toward the
  true outcome as time-remaining shrinks, rather than keeping drift
  neutral until instant resolution.

**For meta category (resolves against real game state):**
- Resolution reads actual state at the deadline (e.g. was DOGO's price
  higher or lower than it was when the market opened) — no random roll
  needed, this is genuinely determined by what happened in the sim.

**Resolution payout:**
- Standard prediction-market payout shape: winning side pays out based
  on the price/odds at time of purchase, same mechanic as the existing
  two example markets already use (`price` field represents cents/odds,
  a winning position pays out at $1 per unit held, losing pays $0) —
  keep this consistent with what's already implemented rather than
  inventing a new payout formula.
- On resolution, generate a ticker headline referencing the actual
  outcome, tying into the existing ticker system (see
  `RETENTION_SPEC.md` section 2, same pattern as live market events
  generating headlines).

## Pricing/odds realism

To feel more like a real prediction market and less like a coin flip
with a price tag:
- Odds should very rarely sit exactly at 50/50 — bias `baseOdds`
  generation toward more decisive-feeling values (e.g. weighted toward
  ranges like 0.15-0.35 or 0.65-0.85) with genuine toss-ups being the
  minority, not the norm — mirrors how real prediction markets rarely
  show true coin-flip pricing on mundane questions.
- Odds should visibly move in response to the same kind of "event" system
  described in `RETENTION_SPEC.md` — a market event or hot-asset
  rotation touching a prediction market's subject should nudge its
  price, not just its own dedicated asset.
- Volume/liquidity flavor (optional, cheap to fake): a small "X traders"
  or "$X in action" label per market, generated plausibly rather than
  tracked precisely — purely cosmetic, adds texture without needing
  real multiplayer data.

## Tier/unlock integration

Per `DESIGN_NOTES.md`, predictions unlock at the existing net-worth
threshold (`TIERS` config, currently 5000). No change to that gate —
this spec is about depth within the tier, not when it unlocks. Once
unlocked, the full pool becomes available for rotation rather than the
two fixed markets currently hardcoded.

## Own nav tab, separate from Markets

Predictions currently live inside the Markets view alongside stocks/
crypto/forex (the "Degenerate markets" list in `renderMarkets()`). Given
the volume of content this spec adds, predictions should move to their
**own top-level nav tab**, separate from both the Markets tab and the
(future) Real Estate tab from `DESIGN_NOTES.md` — not a subsection of
either.

**Current nav** (`index.html`, the `<nav>` element): Portfolio, Markets,
Activity — three `data-view` targets driven by `switchView()`.

**Change:** add a fourth tab, e.g. `data-view="predictions"`, inserted
between Markets and Activity so the order reads as Portfolio → Markets →
Predictions → (Real Estate, once built) → Activity. Follows the same
pattern already in place — a new `<button data-view="predictions"
onclick="switchView('predictions')">` in `<nav>`, a matching `<div
class="view" id="view-predictions">` in `<main>`, no changes needed to
`switchView()` itself since it already works generically off
`data-view`/`id` matching.

**Locked state before unlock:** rather than hiding the tab entirely
before the net-worth threshold is met, show it locked — consistent with
how individual locked asset cards already render (dimmed, lock icon,
"Unlocks at $X net worth" subtext per `renderMarkets()`'s existing
locked-card treatment). Tapping a locked tab should still switch to it
and show that locked explanation full-screen, not silently do nothing —
gives the player visibility into what's coming without letting them
trade early.

**Icon/label:** keep the same short single-letter `dot` style already
used (`$` for Portfolio, `M` for Markets, `L` for Activity) — something
like `?` fits the prediction-market theme and stays visually consistent
with the existing nav rather than introducing a different icon system.

## Notes

- **Recurring fictional characters:** since Gary the pit boss is already
  established in `OPENING_SEQUENCE_SPEC.md` as the opening narrator
  voice's antagonist, reusing him (and other recurring bits — the
  raccoons, the fry stand) across prediction market prompts is a cheap
  way to make the world feel connected rather than every joke being a
  one-off. Worth keeping a small shared "cast" list that both the intro
  sequence and the prediction pool draw from.
- Keep all copy consistent with the dry, self-aware satirical voice
  established in `OPENING_SEQUENCE_SPEC.md` — same "Where this voice
  should echo later" principle applies to prediction market flavor text
  as much as loan shark dialogue or ticker headlines.
- Real-data categories (weather API, anything else pulling from the
  outside world) should be treated as a deliberate v2 decision, not a
  default — the game's entire pricing philosophy up to this point has
  been simulated/fake, and introducing one real-data category is a
  meaningful architectural departure worth deciding on its own, not
  bundling into this pass.

## Build order

1. Move predictions to their own nav tab (see above), still showing the
   two existing hardcoded markets for now — proves the tab/view plumbing
   works before touching content
2. Restructure those two markets into the pool format
   (`PREDICTION_POOL`), no new content yet — proves the data-shape
   change works independent of the tab change
3. Write the actual content pool (25-30 entries across the four
   scripted categories) — this is the highest-effort item and mostly a
   writing task
4. Wire up rotation (6-10 active at once, refreshing on resolution)
5. Add price drift toward true outcome as resolution approaches
6. Add the meta category (resolves against real game state) — build
   this after the scripted categories are working, since it's a
   different resolution code path
7. Optional: volume/liquidity flavor labels
8. Explicitly out of scope for this pass: real external data (weather
   API or similar) — revisit only after the above is live and tested
