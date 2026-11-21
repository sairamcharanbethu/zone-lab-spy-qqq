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

### v3.1 — zero-trade bug, found and fixed (2026-11-20)

v3 compiled cleanly but produced **zero trades** on Nov 13–20, 2026, while v2 produced 4 on the same bars. A Chart GT diagnostic ruled out the two first suspects — the 14:00 entry cutoff and the daily-loss limit — because widening the window to 15:30 and disabling the loss limit still left 0 trades. The real cause was the new risk-based sizing: a 1-minute ATR stop is only ~$0.20–0.40, so $250 of risk divided by a $0.30 stop asks for ~833 shares, about $580k of notional on $25k of equity. Pine's broker emulator enforces the 100% margin limit and rejects oversized orders outright — every entry, silently. The fix caps quantity at what the equity can afford (`math.min(riskQty, equity / close)`, whole shares, minimum 1). Honest consequence: with $25k capital the backtest's risk per trade is capped well below the 1% setting — it measures R-multiples, not dollar risk. To model true 1% risk you would need either more capital or intraday margin (leverage), neither of which this backtest assumes.

## v4 — FTFC + 2-1-2 inside bars (2026-11-20)

Two ideas borrowed from the Stonkstradamus / The Strat indicator ([Stonkstradamus + Inside Bar Pro + FTFC](https://www.tradingview.com/script/UwfCjqZI-Stonkstradamus-Inside-Bar-Pro-FTFC/)), built as an independent implementation in Zone Lab's own code — no code copied from that script:

- **FTFC gate (02 + 03, `useFTFC`, default on):** the last closed candle on 5m, 15m, 60m and Daily must all point the same way before a signal fires — all four green for longs, all four red for shorts, mixed means no trade. Hypothesis under test: the 6-month backtest's losing shorts (22% win rate, −$115.22) were taken against higher-timeframe direction, and FTFC should screen out the worst of them. FTFC state (BULL / BEAR / MIXED) is shown as a new row in the indicator dashboard.
- **2-1-2 inside-bar trigger (02 + 03, `useInsideBar`, default on):** a directional bar, then an inside bar (the pause), then a break of the inside bar. Entry on the break, stop at the other side of the inside bar, target at the directional bar's extreme — defined risk on every trade. Inside-bar signals use their own stop/target (not the ATR stop) in both the indicator's plotted levels and the strategy's exits, and risk sizing is computed from that stop. Marked with a 2-1-2 diamond on the chart and tagged in alerts.
- Both are toggleable inputs so each can be A/B tested against v3.1. The strategy (03) trades exactly the same signal set as the indicator (02), including the new trigger and gate.

Test plan (same as v3.1): compile-check both files on Chart GT, Strategy Tester on Nov 13–20, 2026 (v3.1: 9 trades, +$59.89, PF 2.911), then the 6-month deep backtest (v3.1: −$86.96, PF 0.298 on regular-session-only data — see the caveat above). Note the deep run *can* test the 2-1-2 trigger fairly (it needs no premarket data), but still cannot test the premarket-retest signals. v4 is a hypothesis, not a fix — the numbers decide whether it stays.

**v4 results (2026-11-20, Chart GT): both files compile clean; v4 is WORSE than v3.1 on both runs — the hypothesis failed.** Week Nov 13–20: 4 "trades" but really ONE short position (Nov 17, 10:32 @734.96), +$37.15 vs v3.1's +$59.89 on 9 trades; the 737.783 profit factor is an artifact of near-zero gross loss (−$0.05), not a real edge — the FTFC gate filtered the week down to a single position. 6-month deep (May 21–Nov 20): −$141.04, PF 0.080, win rate 18.75% (3/16), 12 positions vs v3.1's −$86.96, PF 0.298, 46.15%; longs 2/10 (v3.1: 4/4), shorts 1/6 (v3.1: 2/9); only June profitable; commission $58.95 vs gross profit $12.32. The 2-1-2 trigger added positions (12 vs 9) but most lost; the trade list does not label trigger type, so individual attribution is not possible. Verdict: FTFC + 2-1-2 do not improve the system as built. Open: A/B each toggle separately (FTFC-only, 2-1-2-only) on the deep test, or revert v4 defaults to off.

**Revert (2026-11-20):** per the user's call, both v4 inputs now default to **off** in 02 and 03 (`useFTFC = false`, `useInsideBar = false`). With both off, the system behaves exactly as v3.1 — the v4 code stays in place, opt-in, for future A/B tests. Verified on Chart GT: the reverted files reproduce v3.1 exactly on Nov 13–20 — 9 trades, +$59.89, PF 2.911, max drawdown $31.18, win rate 66.67%, trade-for-trade identical.

## v5 — 2-candle story bias + rubber-band trigger (2026-11-21)

Two ideas from the Mathematical Indicator ([roan-maverick/tradingview-mathematical-indicator](https://github.com/roan-maverick/tradingview-mathematical-indicator), MIT, "story, zone, strat" session strategy), implemented independently in Zone Lab's own code — no code copied from that repo:

- **Story bias (02 + 03, `useStoryBias`, default on):** yesterday's daily candle plus the last closed 4H candle. Both green = buy only, both red = sell only. Mixed follows the spec's "Tell the story" rule: at a daily ceiling (not floor) = sell only, at a daily floor (not ceiling) = buy only, otherwise the daily candle (flat daily → 4H). The bias is frozen at the first entry-window bar each day, as the spec freezes it at each session open, and it gates every signal. A Story row (BUY ONLY / SELL ONLY / NONE / OFF) was added to the indicator dashboard.
- **Rubber-band trigger (02 + 03, `useRubberBand`, default on):** a range of candle bodies over the last 10 closed 1-minute bars, then one strong breakout candle that closes beyond it (body ≥ 50% of its own range, size ≤ 50% of the range height, ≥ 2 opposite candles inside the range), then a dip back — a wick ≥ 0.10 × ATR and ≥ 20% of its candle — and a fill when price retakes the breakout extreme within 6 bars. No dip before the break cancels the order ("nipple wick"); a close back inside the range or expiry cancels it too. Entries use defined risk (stop at the breakout candle's far side) and keep the existing 1R scale-out / 2R exit machinery. Breakout (BRK), dip (RB) and fill (RB diamond) markers are plotted on both files.
- Both are toggleable inputs so each can be A/B tested against v3.1. The strategy (03) trades exactly the same signal set as the indicator (02).

**Adaptations from the spec, stated plainly:** the spec searches k = 1..20 for the largest valid range — Zone Lab uses a fixed 10-bar lookback (`rbRangeBars`, tunable 3–20). The spec's daily ceiling/floor are body-based levels built from pivot patterns — Zone Lab uses PDH/PDL with a 0.25 × daily-ATR band. The weekly staircase tiebreaker is omitted (Zone Lab already has 5m trend + VWAP as structure filters). Clean zones, liquidity-grab labelling, and the "come back tomorrow" block are **not** ported — candidates for later A/B tests. The fill is exempt from the entry window (the breakout candle that placed the order was inside it; the fill may land up to 6 bars later). In the backtest the fill executes at the bar close (the spec's `max(open, order price)` fill price is not modelled).

**Why the deep test is fairer for v5 than it was for v4:** neither new idea needs premarket data — the story bias reads daily/4H closes and the rubber band reads 1-minute bodies, both present in the regular-session-only 6-month history. So the deep run tests v5 properly, unlike the premarket-retest signals it still cannot test.

Test plan (same as v3.1): compile-check both files on Chart GT, Strategy Tester on Nov 13–20, 2026 (v3.1: 9 trades, +$59.89, PF 2.911), then the 6-month deep backtest (v3.1: −$86.96, PF 0.298 on regular-session-only data). v5 is a hypothesis, not a fix — the numbers decide whether it stays.

**v5 results (2026-11-21, Chart GT): both files compile clean (after the ta.sum fix); v5 is WORSE than v3.1 on both runs — the hypothesis failed.** Week Nov 13–20: 7 trades, −$60.51 (−0.24%), PF 0.324, max drawdown $82.68, win rate 42.86% (3/7), all 7 short and 0 long, trades Nov 17–20 only — no trades Nov 13 or Nov 16, so the story-bias gate removed v3.1's winners there, including the Nov 18 long (+$8.73 / +$19.95 in v3.1). v3.1 on the same window: 9 trades, +$59.89, PF 2.911 (reproduced exactly by the older tester entry as a control). 6-month deep (May 21–Nov 20, 49,780 bars): 74 trades, −$282.32 (−1.13%), PF 0.516, max drawdown $385.14, win rate 45.95% (34/74), 27 long / 47 short; the rubber band added ~5.7× the trades (74 vs v3.1's 13) with average loss −$14.59 vs average win $8.86 and commission $231.21 against gross profit $301.37. PF 0.516 beats v3.1's 0.298 and v4's 0.080 but is still far below 1.0 — a losing system that trades more often. One small data note: the deep run's Nov 20 trade (entry 10:15 @ $713.91) differs slightly from the chart run's (10:20 @ $712.25); Nov 17–19 matched exactly. Verdict: the story bias screened out winners and the rubber band added losing trades. Open: revert v5 defaults to off (back to v3.1 behaviour, code stays opt-in) or A/B each toggle separately.

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
