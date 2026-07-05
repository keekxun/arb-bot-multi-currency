# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Running the app

No build step — `arb-bot-multi-currency.html` is self-contained (all CSS/HTML/JS in one file). Open directly in a browser:

```bash
open arb-bot-multi-currency.html
```

After any edit, reopen the file or hard-refresh with **Cmd+Shift+R** (a service worker caches the app shell — see below).

`index.html` is a redirect stub (meta-refresh to `arb-bot-multi-currency.html`) so GitHub Pages root serves the app.

## Git workflow

**Claude must commit and push after every meaningful change.**

```bash
git add <files>
git commit -m "concise description"
git push
```

Remote: `https://github.com/keekxun/arb-bot-multi-currency` (`main` branch).

## Companion repo: arb-bot-worker

This app's `PAIRS` config has a **shadow copy** in a separate repo/directory, `~/Desktop/arb-bot-worker` (`worker.js`, deployed to Cloudflare via `wrangler deploy`). That Worker runs the `/thresholds` persistence endpoint (see Data sources below) **and** a per-minute cron job that independently re-fetches Coinbase/DexScreener prices and fires Telegram alerts when a pair's thresholds are breached.

**Any change to a pair here (new pair, new DEX pool, new threshold key) likely needs a matching change in `arb-bot-worker/worker.js`**, specifically:
- `PAIR_CONFIGS[id]` — coinbaseUrl, tokenSymbol, dexPools, buildPairThresholds, buildCexThresholds
- `PAIR_DEFAULTS[id]` — default threshold values
- `PAIR_KV_KEYS[id]` — KV storage key for persisted thresholds
- the `scheduled()` handler's `checkArbForPair(env, id)` list — otherwise the new/changed pair is never cron-checked and never alerts on Telegram

After editing `worker.js`, commit + push that repo too, then run `npx wrangler deploy` from `~/Desktop/arb-bot-worker` to actually push the change live (a git push alone does not deploy it). Verify with `curl "https://arb-bot-worker.keekxun.workers.dev/debug?pair=<id>" -H "X-API-Key: <WORKER_API_KEY>"`.

## Architecture

Single-file, tab-based, multi-pair arbitrage monitor. Currently tracks **XSGD/USDC** and **tGBP/USDC**.

### Pair configuration (`PAIRS` object)

All per-pair behavior is driven by one config object — adding a new pair means adding an entry here, not new code paths:
- `coinbaseBookUrl` / `coinbaseTradeUrl` — Coinbase Exchange level-2 order book endpoint and deep-link for the pair.
- `fxUrl` / `fxLabel` / `fxTransform` — Oanda fxTrade Practice API candle endpoint for the reference spot rate, plus a transform (e.g. XSGD inverts `USD_SGD`, tGBP uses `GBP_USD` directly).
- `dexPools` — array of `{ chain, address, name, display }` pool addresses queried via the DexScreener API.
- `defaultThresholds` — bps thresholds per venue pair, keyed like `polygon_base`, `cex_base`, etc.
- `dexDexThresholds` / `cexDexThresholds` — declarative lists that drive both the dynamic threshold-settings panel and the spread/arb calculations.

`activePairId` tracks the selected tab (persisted in `localStorage`); `switchPair()` swaps config, resets the UI, and restarts the refresh loop.

### Data sources
- `OANDA_TOKEN` — Oanda fxTrade Practice API, used per-pair via `PAIRS[id].fxUrl`.
- Coinbase Exchange public order book (`api.exchange.coinbase.com/products/<PAIR>/book?level=2`) — no key required.
- DexScreener API (`api.dexscreener.com/latest/dex/pairs/<chain>/<address>`) — one fetch per configured pool.
- `WORKER_URL` (Cloudflare Worker at `arb-bot-worker.keekxun.workers.dev`) — persists per-pair thresholds server-side (`GET/POST /thresholds?pair=<id>`), authenticated via `WORKER_API_KEY` on writes. Falls back to `localStorage` if the worker is unreachable.

### Order book walking (`walkBook`)
Takes bid or ask levels and a USDC trade size, returns the effective average USDC/token price for that size. Used to compute `effectiveAsk` (cost to buy) and `effectiveBid` (proceeds from selling) on Coinbase, driven by the adjustable `Coinbase Order Size` input (default 10,000 USDC).

### Arb detection (`checkArb`)
- DEX-to-DEX: compares each pair of DEX pools for the active pair against `dexDexThresholds` (bps).
- CEX-to-DEX: compares Coinbase effective ask/bid against each DEX price against `cexDexThresholds` (bps).
- Opportunities sorted by margin above threshold (most profitable first); alert banner shows buy/sell venue, price, and deep-link to trade.
- Dismissal state (`dismissed[pairId]`) and alert banner are tracked per-pair, so switching tabs doesn't carry over a dismissed/active state from the other pair.

### Thresholds
- Defaults live in `PAIRS[id].defaultThresholds`.
- `loadThresholds()` fetches remote values from the Worker on load (falling back to `localStorage`), `saveThresholds()` writes back to both on any change.
- The threshold panel (`renderThresholdPanel`) is built dynamically from `dexDexThresholds`/`cexDexThresholds` for whichever pair is active.

### Alerts
- Arb banner appears with a beep (Web Audio) when any opportunity clears its threshold, plus a browser Notification if permission was granted (`🔔 Enable Alerts` button).
- Dismiss button suppresses the banner for the active pair until opportunities change.

### PWA / service worker
- `manifest.json` + `icon.svg` make the app installable; `start_url` points at `arb-bot-multi-currency.html`.
- `sw.js` caches the app shell (`arb-bot-multi-currency.html`, `manifest.json`, `icon.svg`) but explicitly bypasses caching for Oanda/Coinbase/DexScreener requests so price data is always live.
- Bump the `CACHE` version string in `sw.js` (currently `arb-multi-v3`) whenever the app shell changes, so installed PWAs pick up the update.

### Refresh loop
- Auto-refresh: 15 / 30 / 60 s (dropdown, default 30 s), independent per browser session via `refreshSecs`. Countdown shown in footer.
- FX and ticker fetches run independently — FX failure shows "Unavailable" without blocking the price table.
