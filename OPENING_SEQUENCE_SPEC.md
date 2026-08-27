# Opening sequence spec — "Rock Bottom"

Companion to `DESIGN_NOTES.md` and `TUTORIAL_SPEC.md`. A short, funny,
scripted intro that plays once before the tutorial, giving the player's
$1,000 starting cash and the whole premise an actual in-fiction reason
to exist instead of just being a number you spawn with. Ends by handing
off directly into Tutorial Step 1.

## Tone

Full self-deprecating degenerate-gambler satire — think the framing of a
guy telling this story to a bartender, half-laughing at himself. Not
edgy/mean, not actually bleak — the joke is the *scale* of his failure
and how matter-of-factly everyone in his life has written him off. Every
beat should be roastable, nothing should read as genuinely grim. If a
line could be read as actually sad rather than funny-sad, cut it.

## Structure

Full-screen sequence, one beat at a time, each auto-advancing after a
hold time OR advanced early with a tap (always let the player skip
ahead beat-to-beat — never force them to sit through the full hold time
if they've already read it). A persistent small "Skip intro" control in
a corner throughout, same as the tutorial's skip pattern.

**Beat anatomy:**
```js
{
  id: 'beat-1',
  visual: 'blackout',       // which background/visual treatment to use
  lines: ['...', '...'],    // one or more lines, revealed in sequence
  holdMs: 2400,             // time before auto-advance if not tapped
}
```

Text reveal: simple fade-up per line (translateY + opacity transition),
staggered ~200ms apart if multiple lines in one beat. No typewriter
effect except for the terminal-boot beat specifically (see Beat 7) where
it's thematically earned — using it everywhere would slow the pacing
down for beats that don't need it.

## Beat-by-beat script

### Beat 1 — Blackout
- **Visual:** pure black
- **Lines:**
  - "You've lost at three casinos this year."
  - "Tonight makes it four."
- **Hold:** 2400ms

### Beat 2 — Casino floor ambience (dim red/gold tint, no imagery needed — just a color wash)
- **Lines:**
  - "Pit boss Gary has personally walked you out twice."
  - "Tonight he didn't even do the polite hand-on-the-shoulder thing."
- **Hold:** 2600ms

### Beat 3
- **Lines:**
  - "He just pointed. From across the room. Security handled the rest."
- **Hold:** 2000ms

### Beat 4 — Text on black, slightly smaller/quieter framing (like a receipt)
- **Lines:**
  - "Lifetime casino losses: unspeakable."
  - "Lifetime casino winnings: a $40 buffet comp, once, in 2019."
- **Hold:** 2800ms

### Beat 5
- **Lines:**
  - "You called your brother. He sent you to voicemail. On purpose. You could hear him do it."
- **Hold:** 2600ms

### Beat 6
- **Lines:**
  - "You applied at the airport fry stand. They said you seemed 'emotionally unavailable for fries.'"
  - "You didn't fight it."
- **Hold:** 2800ms

### Beat 7 — Terminal boot (the technical centerpiece, see below)
- **Visual:** terminal-boot
- **Lines (typewriter, monospace, green-on-black CRT feel):**
  - "> searching for anyone who'll give you money"
  - "> 1 result found"
  - "> loan shark located. terms: catastrophic."
  - "> connecting to DEGENERATE HOLDINGS terminal..."
- **Hold:** 3400ms (typewriter timing drives this beat, not a flat fade)

### Beat 8 — Terminal fully booted, logo reveal
- **Visual:** terminal-boot (continued), "DEGENERATE HOLDINGS" wordmark
  types/glitches in
- **Lines:**
  - "Balance: $1,000"
  - "No casino will take your action. This machine will take anything."
- **Hold:** 3000ms

### Beat 9 — Direct handoff line
- **Lines:**
  - "This is rock bottom. Try not to dig."
- **Hold:** 2200ms, then auto-transitions straight into Tutorial Step 1
  ("Welcome to Degenerate Holdings") — no separate "continue" tap needed,
  this beat *is* the bridge.

## Technical approach

Everything here is CSS transitions + `setTimeout` sequencing — no video,
no canvas, no external animation library, consistent with the
low-resource-use requirement in `DESIGN_NOTES.md`.

**Sequencing engine:**
- A `STORY_BEATS` config array (per above), stepped through by an
  `advanceBeat()` function — same pattern as `TUTORIAL_STEPS`, so this
  can reuse a lot of the same plumbing (index tracking, skip control,
  persisted completion flag).
- Store `introSeen: true` on completion or skip — never replay
  automatically on subsequent app opens. Same "replay from settings"
  affordance as the tutorial is worth adding here too, since testers
  will want to show this to other people.

**Visual treatments (all just CSS on a full-screen fixed container):**
- `blackout` — `background: #000`, plain centered text
- casino-floor beats — same black base with a very subtle radial
  gradient wash (dim red/gold, low opacity) behind the text, not an
  actual illustration — keep this cheap and abstract, the writing is
  doing the work, not the visual
- `terminal-boot` — monospace font, `#0f0`-on-`#000` (classic CRT
  green), a blinking block cursor (`::after` pseudo-element, `steps()`
  animation), and a very subtle scanline overlay (a repeating linear
  gradient at low opacity over the whole container) for texture — this
  is the one beat worth spending a bit more visual effort on since it's
  the transition into the actual app's identity

**Typewriter effect (Beat 7 only):**
- Reveal each line character-by-character via `setInterval` at a fixed
  per-character delay (~28-35ms feels right for this — fast enough not
  to drag, slow enough to read as "typing" rather than just fading)
- Allow tap-to-complete: if the player taps while a line is still
  typing, instantly complete that line rather than skipping the whole
  beat — keeps the skip behavior consistent with "always let them go
  faster," without losing the line's content entirely

**Handoff into tutorial:**
- Beat 9 ending should transition directly into
  `TUTORIAL_STEPS[0]` (the "Welcome" step) with no separate loading
  screen or blank gap — the intro sequence and the tutorial should read
  as one continuous experience, not two separate systems bolted
  together. Practically: the intro sequence component unmounts and the
  tutorial component mounts in the same transition, ideally with a
  shared fade-through rather than a hard cut.

## Where this voice should echo later

Since this establishes a specific narrator voice (bone-dry, matter-of-
fact about disaster, a little affectionate toward the player despite
everything), it's worth reusing that same voice in:
- Loan shark dialogue/flavor text (natural fit — same character energy)
- The broke-modal copy when cash hits zero again mid-game ("Here we go
  again" energy)
- Ticker headlines occasionally breaking from generic market-satire into
  something that references the player's specific ongoing disaster
- The one-time cheat-detection joke response (see `TUTORIAL_SPEC.md` /
  `DESIGN_NOTES.md`) — same dry, unbothered tone

Keeping a single consistent narrative voice across all of these is worth
more than any individual line being clever — it's what makes the satire
feel authored rather than like scattered one-off jokes.

## What NOT to do

- Don't make any beat actually bleak — every line should land as funny-
  pathetic, not genuinely sad. If in doubt, cut toward more absurd/petty
  specifics (the fry stand rejection) rather than anything that reads as
  real hardship.
- Don't reference real casinos, real people, or real brands by name —
  keep "Gary the pit boss," the airport fry stand, etc. fictional and
  generic enough to stay clearly satirical.
- Don't let the intro block replay indefinitely or force full watch-
  through on every session — this plays once, ever, unless explicitly
  replayed from settings.
