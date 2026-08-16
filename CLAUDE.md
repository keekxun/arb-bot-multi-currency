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

This app's `PAIRS` config has a **shadow copy** in a separate repo/directory, `~/Desktop/arb-bot-worker` (`worker.js`, deployed to Cloudflare via `wrangler deploy`). That Worker runs the `/thresholds` and `/order-size` persistence endpoints (see Data sources below) **and** a per-minute cron job that independently re-fetches CEX/DexScreener prices and fires Telegram alerts when a pair's thresholds are breached.

**Any change to a pair here (new pair, new DEX pool, new threshold key, new/non-Coinbase CEX venue) likely needs a matching change in `arb-bot-worker/worker.js`**, specifically:
- `PAIR_CONFIGS[id]` — `cexUrl`, `cexName` (optional, defaults to `'Coinbase'` in `fetchCex`), `tokenSymbol`, `dexPools`, `buildPairThresholds`, `buildCexThresholds`
- `PAIR_DEFAULTS[id]` — default threshold values
- `PAIR_KV_KEYS[id]` — KV storage key for persisted thresholds
- `ORDER_SIZE_DEFAULTS[id]` / `ORDER_SIZE_KV_KEYS[id]` — default order size and its KV storage key
- the `scheduled()` handler's `checkArbForPair(env, id)` list — otherwise the new/changed pair is never cron-checked and never alerts on Telegram

After editing `worker.js`, commit + push that repo too, then run `npx wrangler deploy` from `~/Desktop/arb-bot-worker` to actually push the change live (a git push alone does not deploy it). Verify with `curl "https://arb-bot-worker.keekxun.workers.dev/debug?pair=<id>" -H "X-API-Key: <WORKER_API_KEY>"`.

## Architecture

Single-file, tab-based, multi-pair arbitrage monitor. Currently tracks **XSGD/USDC**, **tGBP/USDC**, **AUDD/USDC**, and **AUDM/USDC**.

### Pair configuration (`PAIRS` object)

All per-pair behavior is driven by one config object — adding a new pair means adding an entry here, not new code paths:
- `cexBookUrl` / `cexTradeUrl` — level-2 order book endpoint and deep-link for the pair's CEX venue. Defaults to Coinbase Exchange for most pairs; AUDM uses BTC Markets since AUDM/USDC isn't listed on Coinbase.
- `cexName` — optional display name for the CEX venue (e.g. `'BTC Markets'`); falls back to `'Coinbase'` if unset. Drives the venue label shown in the price table, spread summary, and threshold panel headings.
- `fxUrl` / `fxLabel` / `fxTransform` — Oanda fxTrade Practice API candle endpoint for the reference spot rate, plus a transform (e.g. XSGD inverts `USD_SGD`, tGBP uses `GBP_USD` directly).
- `dexPools` — array of `{ chain, address, name, display }` pool addresses queried via the DexScreener API.
- `defaultOrderSize` — default CEX trade size in USDC for this pair (varies per pair based on pool liquidity, e.g. thinner pools get a smaller default).
- `defaultThresholds` — bps thresholds per venue pair, keyed like `polygon_base`, `cex_base`, etc.
- `dexDexThresholds` / `cexDexThresholds` — declarative lists that drive both the dynamic threshold-settings panel and the spread/arb calculations.

`activePairId` tracks the selected tab (persisted in `localStorage`); `switchPair()` swaps config, resets the UI, and restarts the refresh loop.

### Data sources
- `OANDA_TOKEN` — Oanda fxTrade Practice API, used per-pair via `PAIRS[id].fxUrl`.
- CEX public order book, per pair's `cexBookUrl` — Coinbase Exchange (`api.exchange.coinbase.com/products/<PAIR>/book?level=2`) for XSGD/tGBP/AUDD, BTC Markets (`api.btcmarkets.net/v3/markets/<PAIR>/orderbook?level=2`) for AUDM. Both are public, no key required, and return the same `{bids, asks}` shape so `walkBook` works unchanged across venues.
- DexScreener API (`api.dexscreener.com/latest/dex/pairs/<chain>/<address>`) — one fetch per configured pool.
- `WORKER_URL` (Cloudflare Worker at `arb-bot-worker.keekxun.workers.dev`) — persists per-pair thresholds (`GET/POST /thresholds?pair=<id>`) and per-pair order size (`GET/POST /order-size?pair=<id>`) server-side, authenticated via `WORKER_API_KEY` on writes. Falls back to `localStorage` if the worker is unreachable. The Worker's own cron job (Telegram alerting) reads both from the same KV store, so changing either in the UI takes effect there too.

### Order book walking (`walkBook`)
Takes bid or ask levels and a USDC trade size, returns the effective average USDC/token price for that size. Used to compute `effectiveAsk` (cost to buy) and `effectiveBid` (proceeds from selling) on the pair's CEX venue, driven by the adjustable `CEX Order Size` input (`PAIRS[id].defaultOrderSize`, currently 5,000 USDC for XSGD/AUDD/AUDM and 10,000 USDC for tGBP).

### Arb detection (`checkArb`)
- DEX-to-DEX: compares each pair of DEX pools for the active pair against `dexDexThresholds` (bps).
- CEX-to-DEX: compares the CEX venue's effective ask/bid against each DEX price against `cexDexThresholds` (bps).
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
- `sw.js` caches the app shell (`arb-bot-multi-currency.html`, `manifest.json`, `icon.svg`) but explicitly bypasses caching for Oanda/Coinbase/DexScreener/BTC Markets requests so price data is always live.
- Bump the `CACHE` version string in `sw.js` (currently `arb-multi-v11`) whenever the app shell changes, so installed PWAs pick up the update. Any new CEX domain added for a future pair needs a matching bypass entry in the `fetch` handler, or the service worker will try to cache stale price data for it.

### Refresh loop
- Auto-refresh: 15 / 30 / 60 s (dropdown, default 30 s), independent per browser session via `refreshSecs`. Countdown shown in footer.
- FX and ticker fetches run independently — FX failure shows "Unavailable" without blocking the price table.
