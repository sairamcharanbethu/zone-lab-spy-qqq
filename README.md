# Zone Lab — SPY / QQQ TradingView Pine Script Project

An independent Pine Script v6 research project inspired by the publicly described feature categories of Trade the Zones. It is not affiliated with that vendor and does not reproduce proprietary source code.

## Files

- `01_trade_the_zones_replica.pine` — feature-category approximation: premarket and prior-day levels, VWAP, 200 SMA, wick rejection markers, and premarket break/retest alerts.
- `02_zone_lab_enhanced.pine` — enhanced rules: failed high/low, breakout/retest, relative-volume filter, VWAP alignment, confirmed 5-minute trend, exhaustion proxy, dashboard, alerts.
- `03_zone_lab_backtest.pine` — strategy version for testing the premarket break/retest setup with ATR stops/targets, date range, one-trade-per-day option, commission and slippage assumptions.

## Intended workflow

1. Open TradingView Pine Editor and paste one `.pine` file at a time.
2. Save and add it to a **SPY** or **QQQ** chart.
3. Use a 1-minute chart. The enhanced and strategy scripts use the last completed 5-minute candle as the trend filter.
4. Start with regular and extended-hours data enabled so the 04:00–09:30 America/New_York premarket range can populate.
5. Use the Strategy Tester only with `03_zone_lab_backtest.pine`.
6. Export trades and record results by ticker, month, market regime, and setup type before changing parameters.

## Important research caveats

- The replica is an independently coded approximation based on feature descriptions, not an exact clone. The paid product's internal formulas and thresholds are not public.
- The enhanced script's exhaustion marker is a heuristic proxy, not a validated exhaustion model.
- The 5-minute filter intentionally uses prior completed 5-minute values to avoid using a still-forming higher-timeframe candle.
- The strategy backtest trades the underlying chart symbol as a share-price proxy. It does **not** simulate 0DTE option contracts, implied volatility, delta/gamma, theta decay, bid/ask spread, liquidity, contract selection, or broker execution.
- No profitability or win-rate claim is made. A script compiling is not evidence of an edge.

## v2 hardening (2026-11-20)

A line-by-line review found five real failure modes, now fixed:

1. **Stale breakout state (01)** — a timed-out breakout never disarmed, so a "retest" could fire hours after its window expired. It now disarms on timeout or on a close back through the level.
2. **Overnight holds (03)** — the strategy had no end-of-day flatten, so a position whose stop/target were untouched was carried overnight and could gap through its stop. All positions now close at 15:55 ET (`eodSession` input).
3. **Backtest didn't test its own system (03)** — it traded retests only, while the indicator (02) also signals failed breakouts. The strategy now trades the same signal set as the indicator, so its results describe the system it claims to test.
4. **No entry window (01, 02, 03)** — signals could fire at 15:59 with no time left for the trade. New `entrySession` input (defaults 09:30–16:00 for indicators, 09:30–15:30 for the strategy).
5. **Silent failure without premarket data (01, 02)** — with extended-hours data off, the levels were blank and no signal could fire, with nothing saying why. A warning label now appears on the chart.

Also: 01 signals are now confirmed-bar only (no intrabar repaint on a 1-minute chart).

Honest limit: none of this makes the system "unstoppable." It removes specific, identified failure modes. The strategy still needs to be compiled in TradingView, backtested out-of-sample, and paper-traded before any real capital — and the backtest still trades the underlying share price, not 0DTE options.

## v3 — 0DTE adaptation (2026-11-20)

The system is now built around 0DTE options as the traded instrument. That changes the design, not just the labels:

- **The backtest measures the underlying move, not option P&L.** Pine cannot price an option — no IV, theta, spread, or pin risk is modelled. A profitable underlying move does not guarantee a profitable 0DTE option. Use the backtest to validate signal timing and direction only.
- **Risk-based sizing (03):** position size is now derived from a fixed risk per trade (default 1% of equity) divided by the ATR stop distance — not a fixed 10% of equity. For the option itself, size by premium risk: decide the dollars you will lose if the option goes to zero, and size contracts from that, not from the underlying notional.
- **Daily loss limit (03):** no new entries once the day is down 2% from the open. A bad day stops being a revenge-trading day.
- **Time stop (03):** a trade still open after 30 minutes is closed. On a 1-minute 0DTE setup, a trade going nowhere is usually wrong — and theta charges rent by the minute.
- **Scale-out (03):** take 50% at 1R, let the rest run to 2R with the stop. All-in/all-out at 2R gave up too many winners that stalled at 1.5R.
- **Earlier entry cutoff (03):** entries stop at 14:00 ET by default. Late-day 0DTE entries face accelerating theta and gamma risk.
- **Confluence score (02):** every signal is scored 0–4 (VWAP alignment, confirmed 5m trend, relative volume, usable premarket range). Signals below the minimum score (default 3) are skipped.
- **Full-plan alerts (02):** dynamic alerts now carry entry, stop, target, and score, so no math is needed mid-trade. Stop/target levels for the latest signal are also plotted and shown in the dashboard.
- **0DTE discipline:** exit options by 15:30 at the latest — do not hold into the close (pin risk, collapsing liquidity). Use 0.40–0.60 delta for the balance of premium and directional exposure; avoid far out-of-the-money lottery tickets on these signals.

Still to prove: full-history out-of-sample results (the Chart GT check ran only ~1 week / 4 trades), separate SPY vs QQQ performance, and paper-trading the option fills before any real capital.

## Validation checklist

- Compile each script in TradingView Pine Editor (Pine Script v6).
- Verify PMH/PML visually against the 04:00–09:30 ET session on several dates, including daylight-saving transitions.
- Check that signals appear only after bar close and do not move after chart reload.
- Test SPY and QQQ separately on 1-minute data across trending, range-bound, high-volatility, and low-volatility periods.
- Compare in-sample and out-of-sample periods; do not optimize on the same dates used for final evaluation.
- Record trade count, net profit, profit factor, max drawdown, average trade, win rate, and consecutive losses.
- Re-run with conservative commissions and slippage. TradingView's broker emulator is an approximation; intrabar fills can differ from actual execution.
- Do not use real capital until the logic is reviewed, paper-traded, and tested against realistic option quotes if the intended instrument is 0DTE options.

## Open-source references

These are references for implementation patterns, not code copied wholesale into this project:

- Premarket Metrics Dashboard: https://github.com/realanthonyc/Premarket-Metrics-Dashboard (source header identifies Mozilla Public License 2.0)
- TradingView Strategies: https://github.com/TradingSignalBot/tradingview-strategies (source header identifies MIT License)

Review the upstream license files and preserve required notices if you adapt upstream code. This project currently contains an independent implementation; attribution links are retained as research references.

## TradingView documentation

- Strategies and broker emulator: https://www.tradingview.com/pine-script-docs/concepts/strategies/
- Other timeframes and lookahead: https://www.tradingview.com/pine-script-docs/concepts/other-timeframes-and-data/
