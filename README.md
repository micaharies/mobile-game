# Degenerate Capital — playtest POC

A single-file, client-only proof of concept covering everything discussed so far:
tiered markets, live price simulation, a headline ticker, a broke/recovery loop
with two minigames (tap-timing and whack-a-mole), a loan shark option with
accruing debt, retirement progress bars, and a portfolio/activity log — all
dark mode, no build step.

## Running it locally

No install needed. Just open `index.html` in a browser, or serve the folder:

```
cd poc
python3 -m http.server 8080
```

Then visit `http://localhost:8080`.

## Deploying on your server

Same pattern as the Corvette site — this is static files, so it drops straight
behind Caddy:

```
degcap.yourdomain.com {
    root * /path/to/poc
    file_server
    encode gzip
}
```

Or containerize with a minimal Dockerfile (nginx or Caddy base image serving
this folder) if you want it in your existing Docker Compose stack.

Because it ships a `manifest.json`, once it's served over HTTPS, phones will
offer "Add to home screen" and it'll run in standalone mode like an installed
app — no app store needed for this playtest phase.

## What's real vs stubbed for this POC

**Working, matches what we designed:**
- Tiered market unlocks by net worth (stocks → crypto → predictions → forex),
  each with a toast + ticker headline fanfare the first time it's crossed.
  Crypto's unlock additionally scripts a one-time pump-then-dump on DOGO so
  the unlock dramatizes itself through the sim instead of a popup.
- Safe havens (stablecoin, index fund, bond) always available
- Live price simulation: random walk per asset with per-asset volatility,
  occasional shock events that also push ticker headlines
- Scrolling ticker mixing dynamic price-shock headlines and a static joke pool
- Broke detection (cash near zero + nothing sellable) auto-opens the recovery sheet
- Three recovery paths: beg (lower payout), community service (better payout,
  random minigame), loan shark (instant cash, accruing debt shown in header)
- Two minigame mechanics wired to the same "session → score → payout" pattern
  discussed (tap-timing, whack-a-mole) — swap in the other three we mocked up
  the same way
- Retirement progress bars against three thresholds

**Deliberately stubbed — this is client-only, no backend yet:**
- Everything lives in `localStorage`. There's no server, no accounts, no
  cross-device sync, and nothing here is cheat-proof — it's for testing feel,
  not security.
- No real-money path of any kind, and no ad integration — that's a separate
  layer to add once the loop itself feels right.
- Real estate/property maintenance isn't in this build yet.
- Retirement "locking" funds into safe havens isn't enforced — it just tracks
  net worth against the thresholds for now.
- The one-time "we caught you" joke response for implausible minigame scores
  isn't wired up, since there's no server to validate plausibility against yet.

## Tuning knobs

Everything you'll want to iterate on while testing lives near the top of the
`<script>` block in `index.html`:
- `TIERS` — unlock thresholds per market tier, plus the ticker `line` shown
  when each is first crossed
- `CRYPTO_LAUNCH_SEQUENCE` — the scripted pump/dump steps applied to DOGO
  when the crypto tier unlocks
- `RETIRE_TIERS` — retirement milestone amounts
- `ASSET_DEFS` / `SAFE_DEFS` — starting price, volatility, and drift per asset
- `HEADLINE_POOL` — the static joke ticker lines
- `tickPrices()` — the price simulation tick, runs every 2.5s
# mobile-game
