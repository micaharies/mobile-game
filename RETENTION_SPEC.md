# Retention & pacing spec

Companion to `DESIGN_NOTES.md`. Addresses a specific playtest finding:
the game feels slow and there's not enough to do once the tutorial ends.
This defines four systems to fix that, roughly in priority order. Build
top to bottom — the offline recap has the highest impact per effort.

## 1. Offline recap (highest priority)

**Problem:** the price simulation only advances while the tab is open, so
there's no reason to reopen the app — nothing happens while it's closed.
This is the single biggest missing retention hook from the original
design (see `DESIGN_NOTES.md`, "Real-time vs. offline simulation").

**Fix, client-only version (no backend needed yet):**
- On every save, store `lastSeenAt: Date.now()` alongside the rest of
  state.
- On load, if `lastSeenAt` exists and is more than ~60 seconds in the
  past, compute elapsed time and **fast-forward the price simulation**
  for that duration before first render — run the same `tickPrices()`
  math in a tight loop (or a closed-form approximation, see note below)
  covering the elapsed ticks, rather than rendering live.
- While fast-forwarding, accumulate a diff: net worth before vs. after,
  and the single biggest mover (asset + percent change) — this becomes
  the recap payload.
- On first render after an elapsed absence, show a recap modal **before**
  the normal UI — not a toast, not a badge. This should feel like the
  main event of reopening the app, not a footnote.

**Recap modal content:**
- Headline stat: net worth change since last visit (up or down, colored
  accordingly)
- Biggest single mover, framed as a ticker-style headline (reuse the
  existing headline voice/tone)
- If the absence pushed cash to zero or below: skip straight into the
  broke flow after the recap closes, rather than showing broke state
  silently in the background
- A single "Continue" action closing the modal into the normal app

**Performance note:** naively running `tickPrices()` once per real
elapsed tick (currently every 2.5s) could mean tens of thousands of
iterations for a multi-hour absence. Cap it — either:
- Run a maximum of N ticks (e.g. 500) at full fidelity and treat that as
  "as far as the simulation goes" regardless of real elapsed time, or
- Use a closed-form random-walk approximation for elapsed time beyond
  that cap (sample the net drift + variance for the full elapsed window
  in one shot rather than tick-by-tick)
Either is fine for a client-only POC; a server-side version later
removes this problem entirely since the tick loop runs continuously
server-side regardless of who's watching.

## 2. Live market events (interrupt-and-decide, not passive drift)

**Problem:** price ticking alone is ambient, not a "thing to do" — it
doesn't ask the player to make a decision in the moment.

**Fix:** periodically surface a short-lived, high-visibility event that
requires an active choice, separate from the ambient ticker headlines.

**Event structure:**
```js
{
  id: 'pump-rumor',
  assetId: 'doge',           // which asset this concerns
  headline: 'Rumor: DOGO about to pump',
  windowSeconds: 45,          // how long the choice is available
  choices: ['Buy now', 'Wait', 'Ignore'],
  resolution: (choice) => { /* apply outcome based on choice + actual sim result */ }
}
```

- Trigger roughly every 2-4 minutes of active play (not while backgrounded)
  — tunable constant, don't hardcode the interval invisibly.
- Present as a dismissible banner/card, not a blocking modal — this is
  meant to add texture during active play, not interrupt it the way the
  broke modal or recap modal do.
- The "correct" choice shouldn't always be the same one — sometimes
  waiting pays off, sometimes acting immediately does, sometimes
  ignoring is correct. If "buy now" is always right, this becomes a
  scripted bonus rather than a real decision.
- Resolution should generate a ticker headline referencing what actually
  happened, tying back into the existing ticker system rather than being
  a separate notification style.

**Short-fuse prediction markets:** alongside the existing prediction
assets (which may resolve over hours), add a rotating one that resolves
in 60-90 seconds during an active session. This gives players something
that pays off *soon* without leaving the screen, which is the main gap
right now — everything currently resolves either instantly (a trade) or
very slowly (holding a position).

**Hot asset rotation:** every few minutes, flag one asset as "hot" —
visually distinct (border glow or badge, not just a color change) and
temporarily amplified volatility. Rotates to a different asset next
cycle. Gives players a reason to keep glancing at the market list rather
than picking one asset and forgetting the rest exist.

## 3. Micro-goals and streaks

**Problem:** the only goals right now are the retirement thresholds,
which are large and abstract this early. Nothing resolves on a
session-length timescale.

**Session challenges:**
- On session start (or on a rolling basis), assign 1-3 small challenges
  from a pool: "make 3 trades," "hit a 20% gain on a single position,"
  "try a market you haven't traded before," "survive one broke event
  without a loan."
- Each has a small cash reward on completion, shown via a toast/small
  banner, not a blocking modal.
- Keep the pool larger than what's shown per session so repeat sessions
  don't feel identical — rotate randomly.

**Win streak counter:**
- Track consecutive profitable trades (sell price > buy price on that
  position). Reset on a loss.
- Surface the current streak somewhere persistent but small (e.g. near
  net worth), with escalating jokey titles at thresholds — e.g. 3 =
  "Hot hand," 5 = "Unstoppable," 8 = "Definitely gonna jinx it," 12 =
  "Certified degenerate." Titles should match the game's existing
  satirical voice — see `DESIGN_NOTES.md` tone notes.
- This is intentionally cheap: it's a counter plus a lookup table, but
  taps the same variable-reward loop that makes this genre sticky.

## 4. Juice — feedback that makes actions feel like they happened

**Problem:** even when something does happen (a trade executes, a price
moves), it may currently just be a number updating with no other signal
— which reads as "nothing happened" even when something did.

- **Price tick flash:** briefly flash an asset's price text or card
  background green/red on the tick where it moves, then fade back to
  neutral over ~400ms. CSS transition, no animation library needed.
- **Trade confirmation:** a brief visual acknowledgment on buy/sell —
  even something like a quick scale-and-fade checkmark or a color pulse
  on the sheet before it closes, rather than just closing instantly.
- **Minigame payout:** already has a result state in the current build;
  make sure the payout number itself animates in (count-up or
  scale-in) rather than appearing instantly.
- **Haptic feedback:** on mobile, trigger `navigator.vibrate()` (where
  supported) on trade confirmation and minigame payout — cheap, and
  meaningfully improves perceived responsiveness on a phone.
- Keep all of this CSS-transition/Web-API based — no new libraries,
  consistent with the low-resource-use requirement in
  `DESIGN_NOTES.md`.

## 5. Debug/testing mode (playtest-only, not player-facing)

Separate from the above — this is a tool for you, not a game feature.

- A hidden or easily-toggled debug flag that:
  - Speeds up the price tick interval (e.g. 500ms instead of 2500ms)
  - Reduces tier unlock thresholds
  - Reduces retirement thresholds
  - Optionally fast-forwards the offline recap simulation artificially
    on demand, so you can test that flow without literally waiting
- Purpose: let a 10-minute playtest session cover more of the game's
  arc than real tuning would allow, so you can evaluate pacing across
  the full loop quickly. Never ship this enabled by default — gate it
  behind a query param, a long-press on a hidden element, or similar,
  clearly separate from anything a real tester would stumble into.

## Build order

1. Offline recap — biggest single fix for "nothing to do," addresses the
   actual retention hook the game was designed around.
2. Juice (section 4) — cheapest to build, immediately makes existing
   actions feel more responsive, no new systems required.
3. Session challenges + streak counter (section 3) — cheap, gives
   near-term goals without new mechanics.
4. Live market events (section 2) — highest effort of the four, biggest
   change to moment-to-moment pacing during active play.
5. Debug mode (section 5) — do this whenever it's convenient; it's a
   tool, not a feature, and doesn't block anything else.
