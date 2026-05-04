# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Running the app

No build step — `arb-bot.html` is self-contained. Open directly in a browser:

```bash
open arb-bot.html
```

After any edit, reopen the file or hard-refresh with **Cmd+Shift+R**.

## Git workflow

**Claude must commit and push after every meaningful change.**

```bash
git add <files>
git commit -m "concise description"
git push
```

Remote: `https://github.com/keekxun/arb-bot` (public, `main` branch).
GitHub Pages: `https://keekxun.github.io/arb-bot/` → redirects to `arb-bot.html`.

## arb-bot.html architecture

Single-file app — all CSS, HTML, and JS in one file.

**Data sources**
- `OANDA_TOKEN` / `FX_URL` — Oanda fxTrade Practice API for SGD/USD spot rate.
- `COINBASE_BOOK_URL` — Coinbase level-2 order book for `XSGD-USDC`. Effective prices are computed by walking the book for a configurable USDC trade size (`TRADE_SIZE_USDC`, default 10k).
- `DEX_POOLS` — Three fixed pool addresses queried via DexScreener API: Uniswap V3 (Polygon), Aerodrome (Base), Uniswap V3 (Avalanche).

**Order book walking (`walkBook`)**
- Takes bid or ask levels and a USDC trade size, returns the effective average USDC/XSGD price for that size.
- Used to compute `effectiveAsk` (cost to buy XSGD) and `effectiveBid` (proceeds from selling XSGD) on Coinbase.

**Arb detection (`checkArb`)**
- DEX-to-DEX: compares each pair of DEX pools against `PAIR_THRESHOLDS` (bps).
- CEX-to-DEX: compares Coinbase effective ask/bid against each DEX price against `CEX_THRESHOLDS` (bps).
- Opportunities sorted by margin above threshold (most profitable first).
- Alert banner shows buy/sell venue, price, and deep-link to trade.

**Thresholds**
- `PAIR_THRESHOLDS`: minimum spread (bps) for DEX-DEX pairs to be considered profitable (accounts for bridge/gas costs).
- `CEX_THRESHOLDS`: minimum spread (bps) for Coinbase vs each DEX chain.

**Alerts**
- Arb banner appears with a beep when any opportunity clears its threshold.
- Dismiss button suppresses the banner until opportunities change.

**Refresh loop**
- Auto-refresh: 15 / 30 / 60 s (dropdown, default 30 s). Countdown shown in footer.
- FX and ticker fetches run independently — FX failure shows "Unavailable" without blocking the price table.
