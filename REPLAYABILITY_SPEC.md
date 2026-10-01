# Replayability spec — runs, builds, and reasons to come back

Companion to `DESIGN_NOTES.md` and `RETENTION_SPEC.md`. Addresses the
playtest finding that the game "gets stale really quick" even after the
retention systems (offline recap, live events, hot assets, challenges,
streaks, flash crash, rug pull, properties) were built. Written to be handed
straight to Claude Code; build order is at the bottom.

## Diagnosis: the idea is fine, the loop has no decisions in it

The premise (satirical degenerate trader, rock bottom, claw back to
retirement) is strong and the tone is already the best part of the build.
What goes flat is structural, and it's three specific things in
`index.html`, not a lack of content:

1. **Every choice is a coin flip with no information.** Prices are a pure
   random walk (`tickPrices()`), so no trade is better than any other in
   expectation. Live-event rumors (`maybeTriggerLiveEvent()`) roll 40% up /
   40% down / 20% flat and show the same headline either way, so "Buy now"
   vs "Wait" can't be reasoned about. Shock headlines are pushed *after*
   the move, not before (despite tutorial step 3 promising "you'll usually
   hear about it here first"). Result: a player can't get better at the
   game, so after ~20 minutes there's nothing left to learn.
2. **There is no run.** One endless save, one net-worth number going up.
   Retirement is three progress bars (`renderRetirement()`) that do
   nothing when reached. Nothing ever ends, so nothing ever restarts
   differently.
3. **Losing doesn't cost anything.** Going broke is a minigame and you're
   back; loan-shark debt drips interest (`accrueDebt()`) with no ceiling or
   consequence. With no stakes, wins don't feel like wins either.

More events, assets, or prediction questions won't fix these; they add
texture to a loop that has no decisions in it. The fix is to give the game
a **run** (start → build → end with a score), make **runs differ**, and let
**skill and knowledge** carry from one run to the next. That's the
roguelite structure, and it fits this premise naturally: every run is a new
rock bottom.

## The proposals

Each item is marked **cheap** (an evening, mostly config + one function)
or **big** (new system, new UI surface, several sessions).

### A. Make decisions readable (skill)

**A1. Rumors with tells — cheap.** Give each live-event template a
`source` with a hidden reliability, and show the source on the banner:
"Gary's cousin" (30% right), "a guy in a Lambo" (50%), "SEC filing leak"
(80%). Bias the outcome roll by the source. Players learn the sources over
runs. Same `MARKET_EVENT_TEMPLATES` shape, one extra field, the roll in
`maybeTriggerLiveEvent()` reads it.

**A2. Foreshadowing headlines — cheap.** Before a shock fires on an
asset, queue it 2–4 ticks ahead and push a vague pre-headline ~60% of the
time ("Unusual volume on BTCX"). Some pre-headlines are fake. Makes the
ticker something you watch rather than wallpaper, and makes tutorial step 3
true.

**A3. Market regimes — cheap.** Each run (and, later, each in-game "week")
has a regime that sets drift/vol multipliers per tier: Bull Run, Crypto
Winter, Meme Mania, Stagflation, Boring Market. Announced on the ticker,
lightly hinted rather than labeled. Reading the regime is the macro skill;
it also kills "always just buy RUGX".

### B. Give the game a run (structure)

**B1. Retirement ends the run — cheap/medium.** Wire the existing TODO:
reaching a `RETIRE_TIERS` amount *in safe havens* lights a "Retire" button.
Retiring locks the run, shows a run summary card (time, peak net worth,
times broke, loans taken, best trade, tier reached), computes a **score**,
and offers "New run". Retiring early at Comfortable vs pushing for Legend
is the big risk decision the design notes already call for.

**B2. Busting ends the run too — cheap.** Give the loan shark a debt
ceiling (e.g. 3 loans or debt > $5k). Past it, the next broke event is
"Vinnie collects": run over, same summary card with a funny epitaph. Begging
and community service stay free and unlimited until then, so the
no-forced-loss/no-IAP ethics hold; it's only the loan path that can end a
run.

**B3. Run length target — tuning.** Aim for a run to take 30–90 minutes of
active play spread over a day or two, so the offline recap still matters.
Current thresholds (Legend at $250k) are probably right; the onboarding
boost should apply only to the first-ever run.

### C. Make runs different (variety)

**C1. Perk draft at each tier unlock — medium. Highest-value item.** When
`checkTierUnlocks()` fires, after the existing fanfare, offer **pick 1 of 3**
perks (non-blocking card, same style as the live-event banner). Perks change
how you play, not just numbers:
- *Insider Uncle* — rumors about one asset are always shown with their true direction
- *Diamond Hands* — positions held 5+ min take half damage from flash crashes
- *Margin Account* — trades can use 2x leverage (and can wipe you twice as fast)
- *Gary Owes You* — one free loan with zero interest
- *Short Seller* — unlocks selling assets you don't own
- *Tax Lawyer* — safe havens yield double drift
- *Influencer* — your buys nudge meme coin prices up a little
- *Day Trader* — challenges pay 3x, but holdings older than 10 min lose 1%/min

  Store perks on `state.run.perks`; each perk is a few lines of code at
  the hook it touches. A pool of ~15 means no two runs have the same build.

**C2. Character backgrounds — cheap.** At run start, pick 1 of 2 random
starting characters, each with a perk and a flaw ("Ex-Crypto Bro: starts
with DOGO unlocked, but $500 cash"; "Retired Dentist: $3,000 cash, can't
buy anything with ☠"). Reuses the intro voice; a short one-line intro beat
instead of replaying "Rock Bottom".

**C3. Rotating asset roster — medium.** Grow `ASSET_DEFS` into a pool
(like the prediction question bank already does) and draw a subset per run:
3 of 6 stocks, 3 of 6 cryptos, etc. Each asset gets a personality (pumps
on weekends, mean-reverts, rugs). Mostly writing work.

### D. Carry-over progression (meta)

**D1. Legacy points + unlocks — medium.** Each run's score converts to
Legacy points. Spending them unlocks **options**, not raw power: new perks
added to the draft pool, new characters, new assets in the roster, a fourth
perk choice, the minigames 3–5 from the design notes. Small power unlocks
are fine (start with +$250), but cap them so run 20 isn't trivial. This is
the "cosmetic carryover" from `DESIGN_NOTES.md` made concrete.

**D2. Achievements that unlock things — cheap.** "Retire without ever
touching a safe haven until the last minute", "Go broke 3 times and still
reach Retired", "Make $10k on a single RUGX position", "Retire with an
active loan". Each one unlocks a perk/character. Gives experienced players
goals that force them to play differently.

**D3. Hall of Fame — cheap.** A local list of past runs (character, perks,
score, how it ended, epitaph). This is where the leaderboard idea starts,
client-only.

### E. Reasons to come back today (return hooks)

**E1. Daily Market — medium. Second-highest-value item.** Once a day, a
seeded run: same regime, roster, character, and event schedule for
everyone (seed = date). Short format (e.g. 15 min of active play or one
real day), one attempt, score saved to the Hall of Fame. Needs
`Math.random()` swapped for a seeded RNG in the sim paths, which is the
real work. Becomes a shared leaderboard once there's a backend.

**E2. Scale challenge rewards — cheap.** `CHALLENGE_POOL` rewards are
flat $30–$80, which is meaningless after the first unlock. Scale to a % of
net worth, or pay Legacy points instead.

**E3. Recap teases the run — cheap.** Extend `showRecap()` with one
forward-looking line: "Rumor mill: something big on BTCX this afternoon"
(seeds A2's next event). Gives the reopen a reason to stay.

## What NOT to change

- Keep the real-time market and offline recap; runs just sit on top.
- Keep at least one free recovery path always available (B2 only caps loans).
- Keep perks/regimes flavored in the dry "Rock Bottom" narrator voice.
- No IAP, still. Legacy points are earned only.

## Build order

**First slice ("the run"), build this first and playtest it:**
1. **B1 + B2** — retirement and loan-shark bust both end the run with a
   summary card and score; "New run" resets state (keep `introSeen` /
   `tutorialComplete`).
2. **C1** — perk draft at each tier unlock, starting with 6–8 perks.
3. **A1** — rumor sources with hidden reliability.

That slice alone turns "number goes up forever" into "this run I drafted
Margin Account + Short Seller and got rugged at $180k", which is the thing
people retell and replay. Everything else layers on cleanly afterwards:

4. A2, A3 (readable markets)
5. D1, D2, D3 (meta progression)
6. C2, E2, E3 (cheap variety and hooks)
7. E1 Daily Market (seeded RNG refactor)
8. C3 rotating roster (mostly writing)

Prompt to give Claude Code for the first slice:

> Read `DESIGN_NOTES.md`, `RETENTION_SPEC.md` and `REPLAYABILITY_SPEC.md`.
> Implement the "First slice" from REPLAYABILITY_SPEC.md (B1, B2, C1 with
> 6–8 perks, A1) in `index.html`. Keep the existing tone, no new
> libraries, put new tuning constants near the existing ones, and add
> debug-mode shortcuts to trigger retirement, a bust, and a perk draft.
