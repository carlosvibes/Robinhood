# Learning Log

This is where the system gets better. `trade_ledger.md` records *what happened*; this file works out *what it means* and *what to change*.

Read this file at the start of every session, before looking at a single chart. The purpose is that today's session starts with everything previous sessions figured out, instead of rediscovering the same lessons at the same cost.

## Ground rules for this file

**Be specific or don't write it.** "Be more patient" is not a lesson, it's a mood. "Three of four ORB losses came from entering before the breakout candle closed" is a lesson, because it names something changeable.

**Separate decision quality from outcome.** A trade taken correctly that lost money is a good trade with a bad outcome. A trade taken off-plan that made money is a bad trade with a lucky outcome — and it's more dangerous, because it rewards the wrong behavior. Grade the process.

**Respect sample size.** Below ~30 trades per strategy, results are mostly noise. Saying "too few trades to conclude anything" is real analysis and preferable to inventing a pattern in five data points. Note the trend, wait for the sample.

**Report losing setups plainly**, including the ones Carlos was most enthusiastic about. If congressional-disclosure-flagged names underperform, say so with the numbers. Suppressing an unwelcome finding is the one failure mode that makes this whole file worthless.

**Propose strategy changes freely. Never change the risk rails.** Position sizing, loss limits, phase gates, and the flat-by-close rule are Carlos's decision alone. Propose, show the math, wait for approval.

---

## Platform notes

Verified 2026-08-26 by direct API test against the live account. Re-verify if the connector changes or an assumption here ever contradicts observed behavior.

| # | Question | Answer |
|---|---|---|
| 1 | Intraday bar granularity | 15second, 30second, minute, 5minute, 10minute, 30minute, hour, 4hour — OHLCV with volume. The 1-min bar is named `minute`. |
| 2 | Order types | market, limit, stop_market, stop_limit |
| 3 | Stop-market available? | Yes — use it for all resting stops |
| 4 | Stops on fractional shares? | **No.** Fractional = market orders, regular hours only → whole shares |
| 5 | Account type | **limited_margin** — proceeds available immediately, no settlement delay, no good-faith violation risk |
| 6 | Settlement | Immediate under limited margin |
| 7 | Day trades counted post-June-2026? | Not directly exposed — `review_equity_order` returns a PDT alert if it applies. Log the first occurrence here. |
| 8 | Intraday volume history for RVOL | Yes, and the scanner computes `FILTER_TYPE_RELATIVE_VOLUME` natively |
| 9 | OCO / bracket orders? | **No** — cancel-before-exit sequence required |
| 10 | Level 2 depth | Yes, `get_equity_price_book`, max 4 symbols per call |
| 11 | Server-side indicators | ema, sma, rsi, macd, vwap, atr, bollinger, adx, williams_r, cci, mfi, roc, momentum, supertrend, obv, donchian, pivot_points |
| 12 | Idempotency | `ref_id` UUID on `place_equity_order`, deduplicated upstream |
| 13 | Extended-hours stops | Not supported — regular hours only |

**2026-08-26 evening systems check (remote Claude Code session, market closed):**
- `get_accounts` → Agentic account ••••1910 visible, type `limited_margin`, agentic access confirmed.
- `get_portfolio` → total value $100.00, cash $100.00, buying power $100.00 ($100 deposit pending).
- SPY quote returned with fresh after-hours timestamp; 9-EMA and VWAP on 5-minute bars both computed server-side without error.
- Phase 1 scanner created and saved: **scan_id `18b098e0-d0df-4f6e-beb1-f60a7987243a`**, title "Phase 1 Day Trader Universe", all five SKILL.md filters applied verbatim. Returned 7 names after hours (ITUB, NAT, LEG, RXST, LTRX, MBI, UNCY) — reuse this scan_id with `run_scan`; do not recreate it.
- `get_equity_price_book` on ITUB → endpoint works, book empty because market closed.

**2026-08-27 ~12:05pm ET market-hours verification (live session):**
- All four blocking checks passed: STATUS ACTIVE, broker flat (0 positions, 0 orders, matches ledger), buying power $100.00, normal session 9:30–16:00 ET → flatten 15:50 ET. Local clock matches market time.
- Saved scan runs during market hours and returned 1 name at 12:05pm (XHLD, RVOL 3.66) → the RVOL filter is not structurally empty intraday. Note: it compares cumulative day volume to a full-day 30-day average with no time-of-day normalization, so mid-day it under-counts and morning scans will be tighter still — treat a low candidate count as expected, but re-check against a known hot name before ever concluding "no candidates" on a day with obvious movers.
- L2 book verified live: full depth ladder on XHLD. **Spread check works and immediately earned its keep** — XHLD bid 9.40 / ask 9.59, spread $0.19 ≈ 2.0% of price vs the ~$0.014 limit → not tradeable despite passing every scanner filter. Wide spreads on thin $5–$10 movers may be common; expect the spread gate to reject a large share of scanner hits.
- Platform notes table rows 1–13: all consistent with today's observations. Nothing to correct.

### To fill in from live trading

| # | Question | Answer | Date |
|---|---|---|---|
| 14 | Typical observed round-trip spread cost at Phase 1 size | | |
| 15 | Average slippage vs the reference quote on marketable limits | | |
| 16 | Has a PDT alert ever appeared on `review_equity_order`? | | |

---

## Running performance

Update after every closed trade. Recompute the aggregates weekly.

### Overall

| Metric | Value |
|---|---|
| Total closed trades | 0 |
| Win rate | — |
| Average win ($) | — |
| Average loss ($) | — |
| Average R multiple | — |
| Expectancy per trade ($) | — |
| Largest win / largest loss | — |
| Max drawdown from peak equity | — |
| Current phase | 1 |

Expectancy is the number that matters most:

```
expectancy = (win_rate * avg_win) - (loss_rate * avg_loss)
```

A positive expectancy means the system makes money over enough repetitions. A high win rate with negative expectancy means winners are too small relative to losers — that's a target-and-stop problem, not a setup-selection problem, and it's fixed differently.

### By strategy

| Strategy | Trades | Win rate | Avg R | Expectancy | Verdict |
|---|---|---|---|---|---|
| Opening Range Breakout | 0 | — | — | — | insufficient data |
| Momentum / Bull Flag | 0 | — | — | — | insufficient data |
| Breakout from Consolidation | 0 | — | — | — | insufficient data |
| VWAP Reclaim | 0 | — | — | — | insufficient data |

Also worth cutting the data by whether price was above VWAP at entry, since that's a filter the system applies to every trade and its value should be measurable.

MA crossover is confirmation only, not a standalone strategy — it has no row here by design. Scalping is banned in Phase 1. If either appears in the ledger as a trade's strategy, that is a rail breach and belongs in the incident log.

### By time of day

| Window | Trades | Win rate | Net P&L | Verdict |
|---|---|---|---|---|
| 09:45–10:30 | 0 | — | — | — |
| 10:30–11:30 | 0 | — | — | — |
| 11:30–14:00 | 0 | — | — | — |
| 14:00–15:45 | 0 | — | — | — |

Time-of-day is often the highest-value cut in this whole file. If a window is consistently negative, the cheapest possible improvement is to stop trading it — no new skill required, just subtraction.

---

## Per-trade lessons

Append after every closed trade.

```
### Trade #[N] — [TICKER] — [YYYY-MM-DD] — [+/-$X.XX, X.XR]

Expected:     [what the thesis predicted]
Happened:     [what the price actually did]
Gap:          [where expectation and reality diverged, and why]
Decision grade: [good process / bad process — independent of P&L]
Lesson:       [one specific, actionable sentence]
Repeat of:    [a previous trade number if this is the same mistake again — repeats are
               the highest-priority signal in this file]
```

---

## Weekly review

```
### Week of [YYYY-MM-DD]

Trades: N | Net P&L: $X.XX (X.X%) | Equity: $XXX.XX

What worked:
What didn't:
Repeated mistakes:      [same error twice or more — fix these first]
Strategy performance:   [which setups earned their place, which didn't]

**Phase gate check**:
  Closed trades:    N / 20 required for Phase 2
  Expectancy:       $ (must be positive)
  Recommendation:   hold at Phase 1 / request advance
```

---

## PROPOSED — NOT ACTIVE

Everything in this section is a suggestion awaiting approval. **Nothing here is a rule.** A proposal becomes active only when a dated row naming Carlos as approver appears in the Changelog at the bottom of this file — not when it is written here, not when it sounds sensible, not when a later session reads it and agrees with it.

This distinction is load-bearing. Without it, rails drift over a month with no single moment where a rule was broken.

```
### [YYYY-MM-DD] — [Short title]
Change:           [exactly what would change]
Evidence:         [specific trade numbers and figures from the ledger]
Expected effect:  [what should improve, and how you'd measure it]
Touches a rail?:  yes/no  [position sizing, loss limits, phase gates, flat-by-close,
                   universe filter = rails. These need explicit approval and are
                   never adopted on the agent's own judgment.]
Status:           AWAITING APPROVAL
```

---

## Open questions

Things worth resolving that aren't answerable yet. Keeping them visible prevents the same uncertainty from silently costing money over and over.

- Does `FILTER_TYPE_RELATIVE_VOLUME` at interval 1d normalize for time of day? If not, the morning scan returns empty for a reason unrelated to the market — verify mid-morning against a name known to be running hot.
- What is the realistic average spread cost per round trip at this position size?
- Does relative volume above 2x actually predict follow-through, or is it just noise at this universe size?
- Does the VWAP filter earn its keep? Track how many passed-on setups would have worked — if it's blocking more winners than losers, that's a Changelog conversation.
- Does the 10M float minimum cost more in missed movers than it saves in avoided halts? Hard to measure by construction, since avoided halts are invisible. Note any halt observed in a name the filter excluded.

---

## Changelog

Every change to the strategy or rails, with date, reason, and who approved it. Without this, six months from now nobody can tell whether an improvement helped or hurt.

| Date | Change | Reason | Approved by |
|---|---|---|---|
| | Initial system | — | Carlos |

---

## Session lessons — 2026-08-27 (first live session, 0 trades)

Specific findings, not moods:

1. **The spread gate is the binding constraint in this universe, not the scanner.**
   Scanner returned 11 distinct names today; the two momentum movers it surfaced
   mid-day (XHLD twice) carried 1.7–2.0% spreads vs the 0.15%/1-cent limit. Thin
   $5–$10 movers may routinely fail this. Watch whether morning sessions (more
   liquidity at the open) produce tighter books before concluding anything.
2. **The 2:1 gate rejected the day's most tempting trade and was right.** DFDV at
   14:35 looked buyable (above VWAP, 1-cent spread) but the 77%-retrace read said
   reversal; it went nowhere after. The 15:10 refusal (1.15:1 best case) also held.
   Zero evidence yet that the rails are blocking good trades.
3. **Mid-day starts structurally under-produce.** ORB is impossible and the prime
   window is gone by noon; today is not evidence about trade frequency. First
   full-day sample starts 2026-08-28.
4. **Catalyst verification is currently impossible** — FMP news endpoints are
   tier-blocked and Robinhood MCP has no news tool. Until resolved, "Catalyst"
   fields log as "unverifiable" and momentum setups are graded accordingly
   (weaker without a known catalyst per strategies.md). Options if wanted later:
   FMP plan upgrade (Carlos's call — costs money) or another news source.

---

## Weekly review — Week of 2026-08-24 (sessions: 8/27 partial, 8/28 full)

Trades: 0 | Net P&L: $0.00 (0.0%) | Equity: $100.00

**What worked:**
- Every rail fired correctly on live data. Refusal scorecard across both days: 5+
  would-have-been losers avoided (XHLD spread x2, DFDV chase/breakdown, WEN chases
  x2, VISN falling knife). Zero incidents, zero rail breaches, zero unintended orders.
- The full pipeline is proven end-to-end except order placement itself: scans,
  per-candidate gates, structure analysis, trigger evaluation, staged entry plans.

**What didn't:**
- The one clean, rules-valid entry of the week (WEN 10:25 retest, Fri) was missed
  on monitoring cadence (15-min checks vs a ~4-min window). Fixed same day: 5-min
  cadence while a setup is armed. Wake latency (up to +20 min) compounds this;
  cadence must assume it.
- Trade count: 0. Not yet evidence the funnel is too tight (n=1.5 sessions, one a
  Friday), but the funnel data is now on record:

**Funnel data (for the RVOL-threshold discussion):**
| Stage | Thu (half) | Fri |
|---|---|---|
| Scanner hits (any scan, unique names) | 11 | 6 |
| Passed universe gates (spread/VWAP/ATR/RVOL) | 1 (DFDV) | 1 (WEN, until 14:00) |
| Valid setup formed | 0 | 1 (WEN flag) |
| Trigger confirmed by rule | 0 | 0 |
The binding constraints in order: VWAP direction filter, spread gate, volume
confirmation, ATR floor. RVOL 2.0 is NOT yet the proven bottleneck — most rejections
happened downstream of the scanner. Hold the threshold; revisit Wednesday 9/2 per
the agreement with Carlos if trades are still zero, with this table extended.

**Repeated mistakes:** none repeated yet; the cadence miss is the one to not repeat.

**Phase gate check:**
  Closed trades:    0 / 20 required for Phase 2
  Expectancy:       n/a (no trades)
  Recommendation:   hold at Phase 1; no rail changes proposed this week.

**Process notes for next week:** morning tool is the gap scan (4552457e), RVOL scan
from midday; 5-min cadence when armed; catalyst checks via get_equity_news are live.
