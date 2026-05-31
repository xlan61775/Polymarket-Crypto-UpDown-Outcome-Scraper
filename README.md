# Polymarket 5-Minute Up/Down Outcome Scraper

A tiny, dependency-light, well-documented reference for reading the **early
"Outcome: Up / Down" signal** of Polymarket's 5-minute crypto up/down markets
(BTC / ETH / SOL / XRP) — *before* the on-chain oracle settles the market.

It is fully standalone: **no trading bot, no broker SDK, no API keys, no
accounts.** Point it at a market, get back `up` / `down`.

> The single source file `polymarket_outcome_scraper.py` is written as a
> tutorial — read the long module docstring at the top for the full
> explanation of *why* this approach is needed and *how* it works.

## Why a browser?

The timely answer is genuinely hard to get any other way:

| Source | Problem |
| --- | --- |
| On-chain oracle (`closed` / `outcomePrices`) | Authoritative, but slow — resolution lags by minutes. |
| Raw HTML (`curl` the page) | The "Outcome: Up" text isn't in the HTML or in any single XHR; it's computed in the browser. |
| Front-end render | Knows the answer within ~20s of candle close. ✅ |

So we *become* the front-end: load the page in headless Chromium, let React
render, and read the outcome out of the live DOM with a small regex.

## The 5 key ideas

1. **Deterministic URL** — `https://polymarket.com/event/{asset}-updown-5m-{t_start}`, where `t_start` is the candle's UNIX open time aligned to a 5-minute boundary (UTC).
2. **Render in headless Chromium** via Playwright.
3. **Poll the DOM inside the browser** with `page.wait_for_function(probe, polling=100)` — resolves the instant the text appears, instead of pulling HTML back to Python on a fixed timer.
4. **Reuse one browser** for the whole process; navigate with `page.goto()` (fast SPA navigation, cached JS/CSS).
5. **Block heavy/irrelevant requests** (images, fonts, media, third-party analytics) but **allow first-party JS/CSS** — block those and React never renders.

Plus: backoff-retry until a deadline, periodic browser recycle to bound memory, and an in-memory cache.

## Install

```bash
pip install playwright
playwright install chromium     # one-time browser download
```

## Use it from the CLI

```bash
# Most recently closed BTC candle:
python polymarket_outcome_scraper.py --asset btc

# A specific candle by UNIX open time:
python polymarket_outcome_scraper.py --asset eth --t-start 1716900000

# Watch the browser (debugging):
python polymarket_outcome_scraper.py --asset btc --no-headless
```

Exit code is `0` on a resolved outcome, non-zero otherwise — convenient for
shell scripting.

## Use it from Python

```python
import asyncio
from polymarket_outcome_scraper import OutcomeScraper

async def main():
    async with OutcomeScraper(asset="btc") as scraper:
        outcome, status, attempts = await scraper.get_outcome_with_retry(
            t_start=1716900000,
        )
        print(outcome)   # 'up' / 'down' / None

asyncio.run(main())
```

## Caveats & ethics

- Reads **public, already-rendered** information. Keep request rates modest and
  respect Polymarket's Terms of Service. Don't hammer the site.
- The front-end label is an **early indicator**, not the final on-chain
  settlement; reconcile against authoritative data if correctness is critical.
- If Polymarket changes its markup, update the regex in
  `_OUTCOME_DOM_PROBE_JS` — that's the one place the parsing lives.

## Need more? (PRO & PTB versions)

This repo is the **minimal, tutorial-style reference**: 5-minute BTC/ETH/SOL/XRP outcome, no dependencies beyond Playwright, read it top to bottom to understand *how* it works. If you've outgrown it, two ready-to-run builds pick up where it stops:

**➡️ [PTB & Outcome Scraper — PRO](https://bluewhalequantlab.gumroad.com/l/polymarket-settled-outcome-up-or-down)** — the full version: **all timeframes (`5m`/`15m`/`1h`/`4h`/`1d`) and all coins** (BTC, ETH, SOL, XRP, DOGE + extendable), with the complete slug factory for the tricky Eastern-Time-aligned `1h`/`4h`/`1d` markets that plain timestamps can't build. Returns PTB, Outcome, and Final Price as a clean object.

**➡️ [T0 Price-To-Beat Resolver](https://bluewhalequantlab.gumroad.com/l/polymarket-t0-price-to-beat-ptb-resolver)** — gets the **opening Price To Beat at T0**, in real time, the instant a window opens (not after settlement). Polymarket's API doesn't expose the PTB for crypto Up/Down markets; this resolves it from the same Chainlink oracle stream and Binance candles the platform settles against. The one piece most Up/Down bots get stuck on.

Both are standalone Python, documented in English + Chinese, with a 14-day fix-or-refund guarantee.

## License

Do whatever you like with it. Attribution appreciated, not required.
