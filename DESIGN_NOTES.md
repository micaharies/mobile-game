# Design notes — Degenerate Capital

Context for anyone (human or Claude Code) picking this project up. This
captures the reasoning behind the mechanics in `index.html`, not just what
they do — so changes stay consistent with the original intent instead of
drifting.

## Premise

Fantasy betting/trading/investing game with fake currency. Satirical tone
throughout — it's poking at finance culture, not simulating it seriously.
No IAP, ever — that's a deliberate ethical stance, not a placeholder. Ad
network (rewarded, opt-in only) is the intended monetization once this
leaves playtest.

## Market design

Hybrid of prediction markets (fast resolution, easy to generate absurd
content, mobile-session-friendly) and crypto-style assets (longer holds,
meme-driven volatility fits the tone). Stocks are the "serious" entry
point new players start with.

**Tiered unlocks by net worth**, not by level or time played — this ties
progression directly to the thing the game is about. Order: stocks →
crypto → predictions → forex → (future) real estate. Each unlock gets a
small fanfare moment (toast + a ticker headline acknowledging it) — this
is the baseline for every tier.

Crypto's unlock goes further: rather than a blocking interstitial, the
first-ever crossing into the crypto tier scripts a one-time pump-then-dump
on DOGO (the meme-crypto "slot machine" asset), synced with matching
ticker headlines. The unlock dramatizes itself through the mechanic
instead of interrupting the session — deliberately in keeping with "ads
should never be interstitial or forced" tone below: nothing here should
block the player from the market they just earned access to. If other
tiers eventually want more than the toast+ticker baseline (predictions,
forex), follow the same pattern: something native to that market's
mechanic, never a modal.

**Safe havens** (stablecoin, index fund, bond) are always available,
never gated. They're intentionally boring — the joke is that most players
will ignore them in favor of chasing volatile plays, which is the
behavior the game is satirizing. Retirement should require net worth
*parked in safe havens*, not just paper gains in volatile positions —
this isn't enforced yet in the POC (see README) and needs wiring up.

## Price simulation

Server-side random walk per asset (drift + volatility + occasional shock
events), not real market data. Each asset has its own volatility profile
— meme crypto should feel like a slot machine, blue-chip stocks should
barely move. This is intentionally cheap to simulate; the "realism" is in
the variance tuning, not in modeling real economics.

## The ticker

Plague Inc.-style rotating headline marquee. Two sources: a static pool of
generic jokes, and dynamic headlines generated from real price-shock
events as they happen. This is meant to be the game's voice/personality —
worth investing writing effort here since it's a lot of the tone.

## Real-time vs. offline simulation

Two different clocks, deliberately:
- **Markets run on real time** — the sim should keep ticking even when
  the app is closed, so re-opening the app produces a "market recap"
  moment (biggest mover, gain/loss since last seen). This is the core
  retention hook — see idle-game patterns (Adventure Capitalist etc.) for
  why the "what happened while I was away" reveal matters so much.
- Chaos over fairness was the explicit choice here — no stop-loss
  protection by default, overnight swings can wipe you out. That's meant
  to feed directly into the broke/recovery loop and keep the satirical,
  unhinged tone.

## The broke mechanic

When cash hits (near) zero and nothing sellable remains, offer three
recovery paths, in increasing "cost":
1. **Beg on the street corner** — lowest effort, lowest payout, always free
2. **Community service** — more effort (a minigame), better payout,
   free — this is the intended slot for an optional rewarded ad to boost
   the payout, never gating it
3. **Loan shark** — instant full reload, but adds debt that accrues
   interest over time, creating self-inflicted future difficulty. No real
   money involved — it's entirely a fake-currency mechanic, but should
   carry narrative/flavor weight (loan shark NPC, escalating "threats" as
   flavor text) since it's basically free story content.

Ads should never be the only way to recover — always at least one fully
free path exists. Ads (when added) should be an optional multiplier/bonus
on top of a free path, opt-in only, never interstitial or forced.

## Minigames

Deliberately varied *mechanics*, not reskins of one mechanic, because a
reskinned version of the same interaction gets stale fast even with new
flavor text. Five designed so far, each a genuinely different feel:
1. Tap-timing bar — stop a moving marker in a zone (precision)
2. Whack-a-mole grid — tap targets before they vanish (reflex/attention)
3. Hold-and-release meter — fill without overshooting (sustained control)
4. Memory sequence — watch and repeat a pattern (recall)
5. Drag-and-sort — sort items into bins before time runs out (spatial;
   needs touch-event rework for mobile, HTML5 drag-and-drop is
   desktop-only)

Architecture pattern for all of them, once server-backed: client renders
the interaction and reports a raw outcome (timing, hits, sequence
correctness) — never a dollar amount. Server recomputes the payout from
that raw data using a per-mechanic payout curve. This is a deliberate
anti-cheat boundary: the client is never trusted to say what it earned.

**Cheat handling (not yet implemented in POC):** first time a submitted
score is implausible, pay out the legitimate capped amount but surface a
one-time joke acknowledgment ("nice try, we saw that impossible reaction
time") — track a per-player `cheat_warning_issued` flag. Every subsequent
implausible submission after that is silently clamped with no message.
The joke should never explain the actual detection thresholds — the goal
is a memorable moment, not a tutorial on how to reverse-engineer the
anti-cheat.

## Real estate (designed, not yet built)

A third asset "shape" distinct from volatile markets and safe havens:
value is mostly stable/appreciating, but generates recurring rent income
that gets interrupted by maintenance events rather than price crashes.

- Maintenance issues should be low-frequency, batched into a single
  non-blocking notification badge (not a modal), and should escalate in
  stages (minor issue → worsens → code violation/rent zeroed) rather than
  being one binary event.
- **Escalation should be tied to login/session count with active
  dismissal, not real-world elapsed time** — a player should only
  progress through the stages after being shown the issue and choosing
  to skip it N times, not just by being away. This keeps "neglect" tied
  to actual decisions rather than punishing players for not opening the
  app (which would fight against the real-time market recap hook).
- Ignoring it long enough should legitimately zero out rent (mirroring
  real legal/code-violation consequences), but resolving it should stay
  low-friction — a one-tap fix, not a second minigame system. Keep the
  minigame-style engagement reserved for the broke mechanic specifically;
  real estate's job is to be a background hum, not competition for focus
  with the core trading loop.

## Retirement / endgame

Multiple tiered thresholds (not one grindy goal) so players can retire
more than once at increasing ambition levels. Retiring should require
funds actually parked in safe havens, creating a real decision point
(cash out a risky win into safety vs. keep pushing). Partial retirement
should be supported — a portion parked/locked while the rest stays in
play — alongside full retirement that locks in a permanent leaderboard
record and resets into a new character, optionally with small cosmetic
carryover perks.

## Visual direction

Bitlife-adjacent: flat cards, minimal borders, no gradients/heavy
shadows, dark mode primary. Monospace numerals for anything
financial/ticker-related to lean into a terminal/statement feel; sans
for UI chrome. Color should stay mostly neutral/dark with small colored
icon chips doing the work of differentiating market types — the palette
should draw the eye to what matters (danger = red, safe = blue/green)
without looking busy. This should stay cheap to render — no canvas, no
WebGL, ordinary DOM/CSS — since low resource use on mobile was an
explicit requirement.

## Technical direction

- PWA-first for alpha/beta, self-hosted (same Docker/Caddy pattern as
  other projects on this server), no app store friction while iterating
- Eventual path to app store release via Capacitor wrapping the same
  React codebase, once the loop is validated
- Backend: Node (market tick loop, websockets for live price push, REST
  for trades), Postgres for durable state, Redis good fit for the hot
  live-tick data specifically
- Client-authoritative data is never trusted for anything that affects
  balance — trades, minigame payouts, debt — server recomputes and
  validates everything

## What's intentionally NOT decided yet

- Exact ad network integration details
- Exact real estate maintenance-event content/flavor text
- Whether forex needs its own distinct mechanic vs. reusing the crypto
  volatility model
- Leaderboard/social feature scope
