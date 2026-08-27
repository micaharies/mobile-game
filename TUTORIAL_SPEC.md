# Tutorial spec — spotlight onboarding

Companion to `DESIGN_NOTES.md`. Defines the actual step-by-step tutorial
content and the engine that drives it, so this can be handed directly to
Claude Code as an implementation target.

## Engine

One reusable spotlight component driven by a config array — not
hand-built overlays per step.

```js
const TUTORIAL_STEPS = [
  {
    id: 'welcome',
    target: null,              // null = centered card, no spotlight cutout
    title: '...',
    body: '...',
    advance: 'button',         // 'button' | 'interaction'
  },
  {
    id: 'net-worth',
    target: '#netWorthVal',
    title: '...',
    body: '...',
    advance: 'button',
  },
  // ...
];
```

**Mechanics:**
- Get the target element's bounding box via `getBoundingClientRect()`,
  draw a focus ring around it (see mockup: `box-shadow: 0 0 0 4px
  var(--bg-accent), 0 0 0 2000px rgba(0,0,0,0.55)` on the target itself
  — this both dims the rest of the screen and rings the target without
  a separate overlay element sitting on top of live UI).
- Tooltip card anchored below the target by default, flips above if
  there's not enough viewport room below.
- Two advance modes per step:
  - `'button'` — a Next button in the tooltip advances immediately
  - `'interaction'` — the step only advances when the player actually
    performs the real action (taps the real buy button, enters a real
    amount) — teaches by doing rather than narrating. Use this sparingly,
    for the 1-2 steps that matter most (first buy, first sell).
- Every step shows a "Skip tutorial" control, always in the same spot,
  always available — never force completion.
- Highlight the **real, live** element, not a screenshot or frozen
  mockup — prices should still be visibly ticking behind the dimmed
  overlay so the tutorial doesn't feel disconnected from the actual app.

**Persistence:**
- Store `tutorialComplete: true` in the same save state as everything
  else once finished or skipped.
- Add a "Replay tutorial" entry in settings/activity view for anyone who
  wants to run it again — resets just the tutorial flag, not the game
  state.

## Step-by-step content

Steps 1-3 are scene-setting (centered cards, no spotlight). Steps 4+
spotlight real elements on the actual portfolio/markets screens.

### 1. Welcome
- **Target:** none (centered card)
- **Title:** Welcome to Degenerate Holdings
- **Body:** You're starting with $1,000 in fake money. Trade it, lose it,
  beg for more, retire rich — all consequence-free. Let's walk through
  the basics.
- **Advance:** button

### 2. Net worth
- **Target:** `#netWorthVal`
- **Title:** This is your net worth
- **Body:** Cash plus whatever you're holding, minus any debt. This
  number is what everything in the game is measured against — market
  unlocks, retirement, bragging rights.
- **Advance:** button

### 3. The ticker
- **Target:** `#ticker-wrap`
- **Title:** Keep an eye on the ticker
- **Body:** Headlines here react to what's actually happening in the
  markets. If something's about to move, you'll usually hear about it
  here first.
- **Advance:** button

### 4. Markets tab
- **Target:** `nav button[data-view="markets"]`
- **Title:** This is where you trade
- **Body:** Tap Markets to see what's available. We'll head there now.
- **Advance:** interaction (tapping the tab both advances the tutorial
  and switches the view — one action does both)

### 5. An asset card
- **Target:** first unlocked asset card in the risky list (e.g. AAPL)
- **Title:** Tap any market to trade it
- **Body:** Every card shows the current price and how much it's moved.
  Tap one to open the buy/sell sheet.
- **Advance:** interaction (must actually tap the card to proceed)

### 6. The trade sheet — amount
- **Target:** `#tradeAmount`
- **Title:** Enter how much to spend
- **Body:** This is in dollars, not units — the game converts it to
  however many shares/coins/units that buys at the current price. Use
  the quick-amount buttons if you don't want to type.
- **Advance:** button

### 7. Buy vs sell
- **Target:** the buy/sell button row in the trade sheet
- **Title:** Buy locks in the trade
- **Body:** Once you own something, this same sheet lets you sell it
  back whenever you want — the price moves in real time, so what you
  paid isn't what you'll get back.
- **Advance:** interaction (must complete a real buy to proceed — this
  is the most important "learn by doing" moment in the tutorial)

### 8. Safe havens
- **Target:** first card in the safe-havens list
- **Title:** Not everything here is chaos
- **Body:** Safe havens barely move. Park money here when you want to
  protect gains instead of risking them — this matters a lot once
  you're going for retirement.
- **Advance:** button

### 9. Tiers
- **Target:** a locked asset card (if one exists at current net worth;
  skip this step entirely if the player has already unlocked everything
  visible)
- **Title:** Riskier markets unlock as you grow
- **Body:** Crypto, predictions, and forex all unlock once your net
  worth crosses their threshold. Grind the boring stuff, unlock the fun
  stuff.
- **Advance:** button

### 10. Going broke
- **Target:** none (centered card)
- **Title:** What happens if you go broke
- **Body:** Run out of cash with nothing left to sell, and you'll get
  three ways back in: beg (fast, low payout), community service (a
  quick minigame, better payout), or a loan shark (instant cash, but
  it'll cost you in interest later). Nobody's coming to bail you out —
  that part's on you.
- **Advance:** button

### 11. Done
- **Target:** none (centered card)
- **Title:** That's the loop
- **Body:** Buy low, panic, sell high, panic differently. Go make some
  questionable decisions.
- **Advance:** button (closes tutorial, sets `tutorialComplete: true`)

## Notes for implementation

- If a target element doesn't exist yet (e.g. player skipped ahead, or a
  step references an element only present after another action), skip
  that step rather than erroring or showing a broken spotlight.
- Step 5 and 7 depend on a specific asset existing/unlocked — fall back
  gracefully to whatever the first available asset is rather than
  hardcoding "AAPL" if the tuning constants change.
- Keep copy in one place (a `TUTORIAL_STEPS` array) so tone/wording can
  be iterated without touching the spotlight engine itself.
- This tutorial should follow the same visual system as the rest of the
  app once that's formalized — same border-radius, spacing, and type
  scale as everywhere else, not a one-off styled overlay.
