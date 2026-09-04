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
| 2026-09-02 | ACTIVE: "Moderate" gate calibration. Reward-to-risk floor 2.0 -> **1.5** (computed from structure; 1.5:1 needs >40% win rate for positive expectancy). ATR(14,5m) floor 0.05 -> **0.04**. Breakout/continuation volume confirmation -> **>=1.6x the avg of the prior 6 completed bars** (supersedes/withdraws the 8/31 ORB-volume proposal). Scanner RVOL filter 2.0 -> **1.6**. NO shorting (short module stays DRAFTED, deferred to Phase 2). ALL RISK RAILS UNCHANGED: $10 max position, $0.25 max risk/trade, whole shares, $5-10 band, stop always resting at broker >=1x ATR and <=3%, long-only above VWAP, spread gate (larger of $0.01/0.15%), float>10M, daily/weekly loss halts, 3-consecutive-loss halt, 6-order cap, flat by close-10. | 4 sessions 0 trades; funnel died at trigger/entry gates, not the scanner. Goal: raise trade frequency to start generating expectancy data at ~neutral expected cost. | Carlos (chat, 2026-09-02) |
| 2026-09-03 | ACTIVE: **Spread gate widened** from max($0.01, 0.15%) -> **max($0.02, 0.35%)** (Lever A). Bounded 10-trade experiment: after 10 closed trades, review expectancy AND spread-paid-vs-gross-P&L; revert if friction is eating the edge. Makes DPRO/BKKT-class movers (0.3% spreads) tradeable; still rejects ABTC-class 0.5%+ junk. ALL OTHER RISK RAILS UNCHANGED. | 6 sessions 0 trades; root cause localized to the spread gate — it rejected 3 of 4 movers on 9/3 and was flagged Day 1. | Carlos (chat, 2026-09-03) |
| 2026-09-03 | ACTIVE: **Trend-follow tactics** (how we trade, not the risk rails). (1) EARLY ENTRY: enter at the first valid structural trigger of a move (pullback-reclaim, micro-consolidation break, first-higher-low) rather than waiting only for the fully-confirmed breakout — provided a tight structural stop keeps risk <=$0.25 and RR>=1.5 to the measured target still holds. (2) TRAILING STOP is the primary exit: once price reaches ~+1R or prints a new higher low, ratchet the resting broker stop up (breakeven first, then trail below successive higher lows or by 1xATR) to ride the trend and lock profit if it reverses. (3) QUICK PROFIT: scalp the move — do NOT default to holding all day; take the trailed profit when momentum stalls. Holding to EOD allowed at operator discretion when the trend is intact. (4) RE-ENTRY permitted if a stopped-out setup re-validates, within the 6-order/day cap and all loss halts. (5) ACT FAST: tighten monitoring to catch and follow trends early. ALL RISK RAILS UNCHANGED (resting broker stop ALWAYS on; trailing = cancel-old-then-place-new-higher; $0.25 risk, $10 position, halts, flat by close-10). | Carlos: "part of the game is get in early and follow the stock up with stop losses... quick money, not hold all day... pick up trends fast and act faster." | Carlos (chat, 2026-09-03) |
| 2026-09-04 | ACTIVE: **Fractional large-cap trading** — the universe/sizing unlock. UNIVERSE expands from $5-10 whole-share stocks to **any highly-liquid stock (fractional shares allowed)** — prefer mega/large-caps with penny-tight spreads (AAPL, NVDA, META, TSLA, AMD, GOOGL, AMZN, MSFT, etc.). Fractional orders are MARKET orders, regular hours only, and **carry NO resting broker stop** (platform note #4), so protection = **MANUAL STOP protocol**: stay attended at <=1-min cadence while in a fractional position and fire a market SELL the instant price trades through the pre-set stop; NEVER leave a fractional position unattended; hard flat by 15:50. SIZING (my implementation per Carlos "as you see fit," conservative): **max position raised $10 -> $25**, **max risk/trade STAYS $0.25** (the real rail, unchanged). Volatility floor re-expressed for large-caps as **ATR% (ATR/price) >= ~0.3% intraday** (replaces the $0.04 absolute floor, which only made sense for $5-10 names). Spread gate for large-caps effectively trivial (they're <0.05%). RETAIN the $5-10 whole-share path as a SECONDARY option (with its resting broker stop) — kept "on the side," not relied upon. UNCHANGED: $0.25 risk/trade, long-only + above VWAP, RR>=1.5, 3-consec-loss halt, $1 daily / $2.50 weekly halt, 6-order/day cap, flat by close-10, NO shorting (Phase 2). MUST verify fractional order mechanics via review_equity_order (preview) before the first live fractional entry (Tue 9/8 pre-market). | 7 sessions 0 trades; ROOT CAUSE = the $5-10 whole-share universe is structurally junk (thin/wide-spread or $10-capped). Fractionals open the entire liquid market (large-cap spreads 0.004-0.03% vs our 0.3-5%) and fix spread/ATR/band/halt at once. | Carlos (chat, 2026-09-04) |

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

### 2026-08-31 — ORB volume confirmation: compare to recent bars, not OR-bar average
Change:           In the ORB/breakout volume confirmation, replace "volume greater
                  than the average of the opening-range candles" with "volume
                  greater than 1.5x the average of the prior 6 completed non-OR
                  bars". All other trigger conditions unchanged.
Evidence:         Two cases in two sessions. (1) WEN 8/28: broke OR high 8.07,
                  never printed a bar above the 206k OR average, ran +$0.11 to
                  8.19+ without us. (2) VISN 8/31 10:05: closed 6.365 above
                  trigger 6.36 on 177k vs 375k required — the 375k average is
                  inflated by the 864k 9:30 bar, which is structurally the day's
                  heaviest. Opening bars carry peak volume; requiring later bars
                  to exceed their average makes the confirmation nearly
                  unreachable after ~10:00, so valid breakouts are refused on a
                  measurement artifact rather than a market signal.
Expected effect:  Breakout confirmations become reachable mid-morning while still
                  requiring genuine volume expansion (1.5x recent). Measured by:
                  confirmed-trigger count and the win rate of resulting trades.
Touches a rail?:  No — this is a strategy trigger definition, not position sizing,
                  loss limits, phase gates, flat-by-close, or the universe filter.
                  Presented for approval anyway per the changelog process.
Status:           AWAITING APPROVAL

**2026-08-31 10:28 addendum to the ORB-volume proposal — counter-case:** VISN's
unconfirmed breakout (10:05, 6.365 close on 177k) failed back to 6.275 by 10:20 —
exactly the staged stop. Under the proposed rule (1.5x prior-6-bar avg = ~100k),
177k would have CONFIRMED and the trade would be stopped for -1R. Scorecard on the
volume gate is now 1 win missed (WEN 8/28) vs 1 loss avoided (VISN 8/31) — a wash.
Recommendation revised: HOLD the current rule; do not approve the proposal yet.
Collect more cases. This is what the evidence process is for.

## Session lessons — 2026-08-31 (Monday week 2, 0 trades)

1. **Coil-compression CASE 4 — the cleanest yet, and the first where ATR alone
   was the blocker.** DPRO coiled 2 hours under 5.38 after a +20% morning run.
   The structure call was right: armed 13:59, trigger fired 15:00-15:05, breakout
   held and extended to 5.45+. Every entry path was rules-invalid: (a) the
   confirmed break was only visible after price reached 5.41+ → no-chase RR 1.31
   FAIL (correct — this gate is 2-for-2 after DFDV); (b) live ATR(14,5m) had
   *fallen* to 0.032 during the coil (quiet bars drag the Wilder average down
   faster than 1-2 breakout bars can lift it; recovery is ~0.002/bar → the 0.05
   floor was unreachable within the entry window). Outcome: paper winner missed
   (retest fill marginal but markable at ~+0.5R unrealized by 15:26, stop never
   threatened).
2. **Compression-gate scorecard (volume + ATR gates on coiled breakouts): 2 wins
   missed (WEN 8/28, DPRO 8/31) vs 1 loss avoided (VISN 8/31).** Not yet
   actionable — VISN's -1R save shows the same gates catch real failures. The
   structural observation for a future proposal: the ATR floor measures *recent
   average* range, so it is lowest exactly at the end of a long coil — the moment
   the setup is best. A floor based on the breakout bar's own range (or ATR
   measured pre-coil) would not have this inversion. NOT proposing yet; logging
   until the case count justifies it.
3. **Pre-committed stand-down criteria work.** The 13:59 ledger block wrote the
   no-chase math and the ATR caution before the trigger; at 15:11 the decision
   was mechanical. Zero temptation-driven errors on a day the candidate ran +21%.
4. **Operationally clean day:** 5-min armed cadence held 13:59→15:11 (12 checks,
   every trigger bar evaluated on time). Today's zero was the rulebook's zero,
   not a monitoring miss.

**Extended funnel table (RVOL-threshold discussion — revisit Wed 9/2 if still 0):**
| Stage | Thu (half) | Fri | Mon |
|---|---|---|---|
| Scanner hits (unique names) | 11 | 6 | 2 (DPRO, VISN) |
| Passed universe gates | 1 (DFDV) | 1 (WEN, until 14:00) | 2 |
| Valid setup formed | 0 | 1 (WEN flag) | 3 (VISN ORB, DPRO flag, DPRO cont.) |
| Trigger fired (structure) | 0 | 1 (missed on cadence) | 2 (VISN 10:05, DPRO 15:00) |
| Trigger confirmed by full rule | 0 | 0 | 0 (VISN vol-fail; DPRO vol marginal) |
| Entry gates passed | 0 | 0 | 0 (DPRO: no-chase + ATR floor) |
Binding constraints this week so far: volume confirmation and ATR floor — both
downstream of RVOL. RVOL 2.0 still not the proven bottleneck: the scanner IS
surfacing the movers (DPRO RVOL 6+); rejections happen at trigger/entry gates.
The Wednesday discussion should weigh trigger-gate calibration, not scanner width.

## Session lessons — 2026-09-01 (Tuesday week 2, 0 trades)

1. **Risk-off tape = correct zero.** Nasdaq -1.4%; the only band entrants were
   damaged goods (a -55% trial-failure knife, a downgrade bounce, a thin-tape
   biotech). The ALMS pass was validated within 90 minutes when its
   "stabilization" base broke and it fell to -58% — the cleanest same-day
   refusal validation yet.
2. **The spread gate earned its keep for the first time.** KYTX had the day's
   only honest structure (above VWAP, flag under 8.27, 2.5:1 geometry) but the
   book was 5-66 shares deep with a $0.02-0.03 spread vs the $0.0124 limit.
   That is not a filter artifact — it is genuinely untradeable liquidity, even
   at 1 share the round-trip friction would have been ~20-25% of the risk
   budget.
3. **ATR-floor compression case count: 5** (WEN 8/28, WEN 8/31 AM, DPRO 8/31,
   VISN 8/31 context, KYTX 9/1 at 0.044). It keeps appearing precisely on
   consolidations. Still logging, not proposing.

**Funnel table extended (Wed 9/2 is the checkpoint):**
| Stage | Thu (half) | Fri | Mon | Tue |
|---|---|---|---|---|
| Scanner hits (unique names) | 11 | 6 | 2 | 5 (DPRO, MNSO, ALMS, IHS, KYTX) |
| Passed universe gates | 1 | 1 | 2 | 0-1 (KYTX failed spread; rest failed RVOL/VWAP/thesis) |
| Valid setup formed | 0 | 1 | 3 | 1 (KYTX flag) |
| Trigger fired (structure) | 0 | 1 | 2 | 0 |
| Trigger confirmed by full rule | 0 | 0 | 0 | 0 |
| Entry gates passed | 0 | 0 | 0 | 0 |
Four days, 0 trades. The scanner surfaces movers fine; the funnel dies at
trigger/entry gates (volume confirmation, ATR floor, spread on thin names,
no-chase RR). The Wednesday proposal to Carlos should present exactly this:
the RVOL threshold is NOT the bottleneck, so widening the scanner would add
candidates of the type already being refused. The honest options are (a) keep
waiting — the rails are behaving correctly and the tape has been genuinely
poor (risk-off week, no clean momentum in the band), or (b) calibrate the
trigger gates with evidence (the ORB-volume proposal already on file, plus the
ATR-floor compression pattern at 5 cases). Present both; recommend nothing
until the numbers justify it.

**2026-09-01 evening — Owner directive from Carlos (chat):** After Wednesday's
session, lower parameters by ~20%, with the specific gates left to my judgment
based on the evidence. Also: confirmed the operating vision (continuous
monitoring, take the best catalyst/volatility name when it confirms, frequent
in-position checks, stops to lock quick wins — all current behavior), and
requested a shorting/"bet against" capability for bearish tapes. Plan of
record: Wednesday pre-market I write the full calibration proposal (ATR floor
0.05->0.04, volume confirmation 2.0x->1.6x, scanner RVOL 2.0->1.6; spread
gate, RR>=2/no-chase, and all risk rails UNCHANGED) plus a drafted short
module (recommendation: hold until Phase 2 / 20 closed trades), both under
PROPOSED - NOT ACTIVE. Activation still requires Carlos's dated Changelog row
naming the specifics — his 20%-ish chat directive authorizes preparing the
change, and his sign-off on the exact values completes it.

### 2026-09-02 — Gate calibration (~20% looser), per Carlos's directive
Change:           Three trigger/entry gates loosen by ~20%; everything else
                  unchanged.
                  1. ATR(14,5min) floor: $0.05 -> $0.04.
                  2. Breakout/continuation volume confirmation: "2x recent" /
                     OR-candle-average -> ">= 1.6x the average of the prior 6
                     completed bars" (uniform basis; this SUPERSEDES the
                     2026-08-31 ORB-volume proposal, which is withdrawn and
                     folded in here).
                  3. Scanner RVOL filter: > 2.0 -> > 1.6.
Explicitly UNCHANGED: spread gate (validated by KYTX 9/1 — real friction),
                  RR >= 2.00 from structure + no-chase max fill (2-for-2
                  saving us from confirmed losers: DFDV, DPRO 8/31 chase),
                  VWAP direction filter, float > 10M, $5-10 band, stop >= 1x
                  ATR and <= 3%, and ALL risk rails (position cap, $0.25
                  risk, loss halts, 6-order cap, flat by close-10).
Evidence:         4 sessions, 0 trades; funnel dies at trigger/entry gates,
                  not the scanner (see funnel table above). ATR-compression
                  pattern: 5 documented cases where the floor was lowest
                  exactly at the end of a coil.
Honest replay of the new gates against the logged record:
                  - VISN 8/31 ORB: 177k = ~1.77x prior-6 -> WOULD have
                    confirmed under the new rule -> -1R loser TAKEN (-$0.25).
                  - WEN 8/28 ORB: would likely have confirmed on the break ->
                    probable winner taken (missed +$0.11+ run).
                  - DPRO 8/31: STILL blocked — ATR was 0.032 < 0.04 and the
                    only confirmed fill was chase-priced. The calibration
                    does NOT capture that one; only a floor <= 0.03 would,
                    and that is too loose to defend.
                  - KYTX 9/1: ATR 0.044 passes 0.04, but spread still fails
                    -> still no trade (correctly).
                  Net on the replayed sample: roughly one loser and one
                  winner added — approximately breakeven P&L, but ~2 trades
                  instead of 0. The honest framing: this calibration mainly
                  RAISES TRADE FREQUENCY so the system starts generating the
                  data the whole project exists to collect, at close to
                  neutral expected cost. It is not a promise of more wins.
Expected effect:  1-3 trades/week instead of 0. Measured by: trades taken,
                  win rate, expectancy per trade, and whether refused setups
                  still validate (the refusal scorecard continues).
Touches a rail?:  No risk rail changes. Trigger definitions + scanner width
                  only. Presented for approval per the changelog process.
Status:           ***APPROVED & ACTIVE 2026-09-02*** — Carlos chose the
                  "Moderate" package in chat, which ALSO lowers the RR floor
                  2.0 -> 1.5 (bigger lever than the original ~20%). See the
                  Changelog row dated 2026-09-02. Now the live ruleset.
Changelog row to add on approval:
                  | 2026-09-02 | Carlos | Gate calibration: ATR floor 0.04,
                  volume confirm 1.6x prior-6 bars, scanner RVOL 1.6.
                  Supersedes 8/31 ORB-volume proposal. All risk rails
                  unchanged. |

### 2026-09-02 — Short module (bet-against capability) — recommend HOLD until Phase 2
Change:           Add a SHORT setup class, mirroring the long playbook:
                  - Setups: breakdown below VWAP only (mirror of the VWAP
                    long filter), momentum-breakdown and failed-bounce
                    continuation; same structure-derived targets, RR >= 2.00.
                  - Risk identical: $0.25 max risk, $10 max position, whole
                    shares, 1 short position max at a time, counts toward
                    all loss halts and the 6-order cap.
                  - Protection: a resting BUY-stop-market placed immediately
                    after fill (mirror of the long stop), stop distance >= 1x
                    ATR and <= 3%.
                  - Exclusions (stricter than longs): no float < 25M, no
                    stocks that have been halted today, no names down > 30%
                    on the day (halt/squeeze risk), no shorting into the
                    first 15 minutes, flat by close-10 as always.
Why HOLD:         (1) A short's loss is unbounded and gaps/halts make the
                  stop unenforceable at any price — the one risk class the
                  $100 book cannot absorb; our high-RVOL universe is exactly
                  where squeezes and halts concentrate (ALMS bounced +5.8%
                  overnight after the -58% day — that gap is the short's
                  overnight nightmare, and intraday versions exist too).
                  (2) Zero closed trades: the long system's edge is still
                  unmeasured. Adding a second direction now doubles the ways
                  to lose and halves the sample per direction. The 20-trade
                  Phase 2 gate exists for exactly this.
Status:           DRAFTED — recommend activation be revisited at Phase 2
                  (20 closed trades, positive expectancy). Carlos may
                  activate earlier via changelog row; the rails above would
                  apply from day one.
Changelog row to add on approval:
                  | YYYY-MM-DD | Carlos | Short module active per 2026-09-02
                  draft: VWAP-breakdown shorts, $0.25 risk, resting buy-stop
                  mandatory, exclusions as drafted. |

## Session lessons — 2026-09-02 (Wednesday week 2, 0 trades; RULES CHANGED midday)

1. **The Moderate calibration went live and immediately worked as designed.**
   Within ~1hr of activation it produced the FIRST arm-able long of the whole
   project (OABI VWAP-reclaim, RR 1.67 — would have been refused under the old
   2:1). It just didn't trigger (OABI broke down). The rules reached a live
   entry gate that the old rules never let us reach. That's the unlock.
2. **DPRO cleared the ATR floor for the first time (0.056)** — Monday's
   volatility expansion healed the compression, exactly as the ATR-compression
   evidence predicted. Its ORB triggered but the confirmation and the run came
   in the same 5-min bar, leaving no non-chase entry; then the breakout FAILED
   (2 closes below VWAP, flushed to 5.48). A would-be 5.55 entry = -1R.
3. **Process error (logged, honest): the 10:06 DPRO over-caution.** My
   pre-written criteria (pullback <=5.55 + above VWAP + spread) were MET at
   10:06, but I added an unwritten "wait for a confirming close" condition at
   the decision point and skipped it. FIX: when pre-committed mechanical
   criteria are met, EXECUTE — do not add conditions mid-decision. (This session
   the over-caution avoided a loss, but that's luck, not vindication of the
   slip; the discipline point stands either direction.)
4. **Spread + ATR gates keep earning their keep:** VSTM +5% but ATR 0.026 (dead-
   money grind, correctly skipped); GBTG 10x RVOL but price PINNED (block-cross/
   arbitrage, no trend); QMLS a decliner. The scanner surfaces movers; the
   entry gates correctly reject the un-tradeable ones.
5. **Usage-limit gap (~1hr midday):** account-wide limit (all Claude sessions
   share it), not this chat alone. Broker-side stops make it a monitoring gap,
   not a capital risk. Mitigation now standing policy (below).

**Funnel table extended:**
| Stage | Thu | Fri | Mon | Tue | Wed |
|---|---|---|---|---|---|
| Scanner hits (unique) | 11 | 6 | 2 | 5 | 5 (DPRO,ALMS,OABI,QMLS,GBTG,VSTM) |
| Passed universe gates | 1 | 1 | 2 | 0-1 | 2 (DPRO,OABI) |
| Valid setup formed | 0 | 1 | 3 | 1 | 2 (DPRO ORB, OABI reclaim) |
| Trigger fired | 0 | 1 | 2 | 0 | 1 (DPRO 9:50) |
| Confirmed + enterable | 0 | 0 | 0 | 0 | 0 (DPRO chase; OABI broke down) |
| Entry gates passed | 0 | 0 | 0 | 0 | 0 |
5 sessions, 0 trades — but Wed is the first day a setup ARMED under enter-able
geometry (post-calibration). The block is now stock follow-through, not the
rulebook. Thursday runs on the new gates from the open.

## Standing policy — usage/token discipline (2026-09-02)
- Idle/flat + nothing armed: 20-30 min cadence (NOT 15).
- Tighten to 5-min ONLY when a setup is armed near its trigger or in-position.
- Do NOT pull the full L2 price book except in the final seconds before an
  actual entry (it is a large payload). Use quote + bars + VWAP + ATR for
  routine reads.
- Keep ledger ticks terse; batch data pulls. One validated action beats many
  speculative checks.
- The account-wide usage limit is shared across ALL of Carlos's Claude sessions;
  a higher plan tier (Max) is the only way to raise the ceiling — that's his to
  decide.

## Session lessons — 2026-09-03 (Thursday week 2, 0 trades; first FULL day on Moderate gates)

1. **The spread gate is now the confirmed primary blocker — and it was flagged
   Day 1.** Today 3 of 4 evaluated movers failed on spread alone: ABTC $0.05
   (0.53%), BKKT $0.03 (0.36%), DPRO rewidened to $0.02 (0.31%) — all vs the
   max($0.01, 0.15%) gate. This is the SAME finding as the 2026-08-27 session
   lesson #1 ("spread gate is the binding constraint in this universe, not the
   scanner"). One week, same wall. It is no longer a hypothesis. Formal proposal
   below.
2. **The one tight-spread mover (SBET, $0.01) failed on ATR, not spread —
   revealing the real tension.** Liquid names (SBET 216M float, BULL 448M) carry
   naturally tight spreads BUT move less / coil (SBET ATR 0.034 after a 2.5h
   consolidation). Thin names (ABTC/DPRO/BKKT, 23-37M float) move hard but carry
   0.3-0.5% spreads. The universe splits into "moves but can't execute" vs
   "executable but doesn't move enough." THIS is the structural problem, stated
   precisely. Neither the RR nor ATR loosening from the Moderate package can fix
   it — it's a spread-vs-liquidity tradeoff, not a trigger-tightness problem.
3. **Arm discipline held cleanly.** DPRO armed on a day-high breakout with a
   "break AND HOLD >6.39" trigger. It poked 6.4155 intrabar (on 70k vol) but
   CLOSED 6.3486 and faded to 6.295. Requiring the HOLD (not the poke) correctly
   avoided a wick-chase that would have been immediately underwater at the 6.28
   stop. Note this does NOT contradict the 9/2 "execute pre-written criteria"
   lesson — the pre-written criterion WAS break-and-hold, and hold never
   happened. Executing the actual criterion = not entering.
4. **Usage discipline worked** — no rate-limit gap today (vs Wed's ~1hr). The
   20-30/5-min cadence policy held budget through a full session.

**Funnel table extended:**
| Stage | Thu(8/28) | Fri | Mon | Tue | Wed | Thu(9/3) |
|---|---|---|---|---|---|---|
| Scanner hits (unique) | 11 | 6 | 2 | 5 | 5 | 8 |
| Passed universe gates | 1 | 1 | 2 | 0-1 | 2 | 4 (DPRO,ABTC,BKKT,SBET) |
| Valid setup formed | 0 | 1 | 3 | 1 | 2 | 2 (DPRO break, SBET coil) |
| Trigger fired | 0 | 1 | 2 | 0 | 1 | 1 (DPRO poke, no hold) |
| Confirmed + enterable | 0 | 0 | 0 | 0 | 0 | 0 |
| Entry gates passed | 0 | 0 | 0 | 0 | 0 | 0 |
6 sessions, 0 trades. Thu(9/3): the funnel reached the ENTRY-GATE stage on 4
names — the widest funnel yet — and died there, 3x on SPREAD, 1x on ATR. The
block has migrated from "no setup" (early week) to "setup exists but fails the
spread/ATR execution gates" (now). That precisely localizes the fix.

### 2026-09-03 — PROPOSAL: spread-gate calibration (PROPOSED — NOT ACTIVE, needs Carlos's dated Changelog row)
Problem:          6 sessions, 0 trades. Root cause now localized to ONE gate:
                  the spread limit max($0.01, 0.15%) rejects essentially every
                  volatile sub-$10 mover (they trade 0.3-0.5% spreads), while the
                  only tight-spread names are too liquid to move/clear ATR. This
                  is the Day-1 finding, now confirmed across the full week.
Two candidate levers (Carlos picks; both are rulebook changes):
  LEVER A (spread tolerance) — widen the gate from max($0.01, 0.15%) to
                  **max($0.02, 0.35%)**. Catches DPRO (0.31%), BKKT (0.36% ~edge),
                  and DPRO-class $0.02 names; still rejects ABTC-class 0.5%+ junk.
                  COST (honest): friction rises from ~$0.012 to up to ~$0.03
                  round-trip on a ~$8 / 1-share position. Against a $0.25 max risk
                  and typical $0.15-0.25 target, that's ~10-15% of gross reward
                  paid to spread. On an UNPROVEN edge (0 closed trades) this is
                  real: it could bleed the account faster if the setups don't win.
                  Bounded as a 2-week / 10-trade experiment with a hard review.
  LEVER B (universe liquidity) — instead of paying spread, raise the scanner's
                  liquidity floor (e.g. avg vol > 3-5M or float > 50M) so it
                  surfaces liquid movers (SBET/BULL-class) that carry $0.01
                  spreads naturally. COST: those names move less in % and coil
                  more (SBET failed ATR today), so fewer big-range setups — may
                  just move the block from "spread" to "ATR/RR." Lower capital
                  risk, possibly still 0 trades.
Recommendation:   **LEVER A, bounded**, as the direct test of Carlos's stated
                  goal (take small volatility profits on the names that actually
                  move). Rationale: his trading style (quick in-and-out on 10-20%
                  movers with $0.10-0.30 intraday swings) can absorb a $0.02-0.03
                  spread when the move dwarfs it; and the ONLY way to get the
                  expectancy data the project needs is to actually take trades in
                  the tradeable universe. Pair it with a strict stop: if after 10
                  trades expectancy is negative and spread friction is the driver,
                  revert. Keep ALL risk rails ($0.25 risk, $10 position, loss
                  halts, resting stop) UNCHANGED — this only touches the spread
                  filter's threshold.
Touches a rail?:  YES — the spread gate is a risk rail (protects against friction
                  bleed). This is why it routes to Carlos, not self-activated.
                  The 2026-09-02 package explicitly left it unchanged; this
                  revisits that specific decision with a week of confirming data.
Measured by:      trades taken, win rate, expectancy/trade, and a new column:
                  spread paid vs gross P&L per trade (to see if friction is
                  eating the edge).
Status:           PROPOSED 2026-09-03. Awaiting Carlos. NOT active until he adds a
                  dated Changelog row naming the exact threshold.
Changelog row to add on approval (Lever A):
                  | 2026-09-03 | Carlos | Spread gate widened max($0.01,0.15%) ->
                  max($0.02,0.35%) as a bounded 10-trade experiment. All other
                  rails unchanged. Review expectancy + spread-vs-P&L after 10
                  trades. |

## Playbook — Trend-follow tactics (ACTIVE 2026-09-03, Carlos directive)

The shift: from "wait for full breakout confirmation, take one fixed-target trade"
to "get in EARLY on a valid trigger, TRAIL the stop up to ride the move, take
QUICK profit, RE-ENTER if it re-sets." Higher activity, higher variance, tighter
management. Risk rails are UNCHANGED — this changes entry timing and exit
management, not how much we can lose per trade.

**1. Early entry (earlier trigger, same risk cap).**
- Valid early triggers (any one), all still LONG + above VWAP:
  a) Pullback-reclaim: price pulls back to rising VWAP / prior breakout level /
     a higher low and reclaims it on a green bar.
  b) Micro-consolidation break: a 2-4 bar tight flag breaks up (don't need the
     full-range breakout).
  c) First higher-low after an impulse: enter on the turn off the higher low.
- STILL REQUIRED at entry: tight structural stop just below the trigger low so
  risk <= $0.25 (1 share => stop <= $0.25 below entry AND <=3% AND >=1xATR);
  measured-move target giving RR >= 1.5; spread <= max($0.02, 0.35%); volume
  showing up (>=1.6x prior-6 on the trigger bar preferred, but an early
  pullback-reclaim may precede the volume surge — use judgment, log it).
- Early entry usually gives a TIGHTER stop (closer support) => better RR, but
  LOWER hit-rate (less confirmation). Net expectancy is the open question the
  experiment measures.

**2. Trailing stop = primary exit (the core mechanic).**
- Initial: place the resting stop-MARKET immediately after fill, as always.
- Ratchet rules (move the stop UP only, never down):
  - At ~+1R unrealized (or the first new higher low): move stop to BREAKEVEN
    (entry). Now it's a free roll.
  - Thereafter: trail the stop below each successive higher low (structure trail),
    or ~1xATR below price if the move is vertical with no clean pullbacks.
  - Keep the trail loose enough to survive normal pullbacks (below the higher
    low, not at it) — a too-tight trail whipsaws out before the real move.
- Mechanic (1 share, can't hold two stops): to raise the stop -> cancel the
  resting stop, VERIFY cancellation by query, then IMMEDIATELY place the new
  higher stop with a fresh ref_id, then verify it rests. Brief unprotected gap
  is acceptable given active monitoring; do it on a pullback, not mid-spike.
- Exit = the trail gets hit (locks the trailed profit) OR momentum clearly
  stalls (take it manually via cancel-verify-then-sell) OR flat by 15:50.

**3. Quick profit, not buy-and-hold.**
- Default is to SCALP the move and bank the trailed gain when momentum stalls
  (lower highs, volume dying, VWAP loss). Do NOT default to holding to EOD hoping
  it closes high.
- Holding longer is allowed at operator discretion ONLY when the trend is clearly
  intact (higher highs + higher lows, volume sustaining) — and the trailing stop
  protects it either way.

**4. Re-entry.**
- If stopped out but the setup RE-VALIDATES (reclaims the trigger, prints a new
  higher low with volume), re-enter — same full entry sequence, fresh ref_ids.
- Budget: each entry burns 1 of the 6 daily opening orders. The 3-consecutive-
  loss halt and daily/weekly $ halts still bind. Don't revenge-trade a chop.

**5. Act fast.**
- In a live move, tighten monitoring (1-2 min) to catch the early trigger and to
  manage the trail. Usage-discipline still applies when flat/nothing armed
  (20-30 min), but a live trend justifies the faster cadence — that's the trade,
  not idle polling.

**Honest note on this package + the wider spread gate:** together these
materially raise trade frequency AND variance. More entries = more spread paid
and more false starts. That's the intended experiment (finally generate
expectancy data), but it means the account can now actually move — down as well
as up. The $0.25/trade cap + halts bound the damage; the 10-trade review is where
we find out if early-entry trend-following clears its costs.

## 2026-09-04 — PROPOSAL: signal-source expansion (Carlos directive) — legality-sorted
Carlos: add (1) copying successful/profitable day traders, (2) watching media
from political figures / people of power, (3) "looking out for insider trading."
These are CANDIDATE-GENERATION / CATALYST layers — they feed the SAME risk-gated
execution framework (rails unchanged). Sorted by feasibility + legality:

GREEN (legal, feasible, high-value) — implement:
- POLITICAL / MEDIA CATALYSTS: trade news-driven moves from policy/political
  figures. Tools available: Robinhood get_equity_news, FMP news, WebSearch.
  Use to (a) explain WHY a scanner mover is moving (catalyst grade), (b) surface
  sectors/names in play after a major policy statement. Feeds candidate list +
  catalyst confirmation; does NOT bypass entry gates.
- CONGRESSIONAL TRADE DISCLOSURES (STOCK Act): U.S. senators'/reps' trades are
  PUBLICLY disclosed by law. Tools: FMP `senate` (+ house) endpoints. Legal,
  well-known "follow the smart money" signal. Use as a watchlist/candidate
  source (names disclosed as bought by multiple members = a research lead), then
  the name must still pass ALL our entry gates intraday. NOT a blind copy.
- CORPORATE INSIDER DISCLOSURES (SEC Form 4): company insiders' OWN buys/sells,
  publicly filed with the SEC. Tools: FMP `insiderTrades`, Robinhood
  get_sec_filing(_facts). Cluster-buys by execs = a legitimate public bullish
  signal. Legal. Candidate source only; entry gates still apply.

AMBER (feasible but limited) — approximate, set expectations:
- COPYING DAY TRADERS: there is NO clean API for "what profitable day traders
  are buying now." No social-sentiment (StockTwits/Twitter) tools in this
  session. Best approximation: unusual-volume + our momentum scanners + options
  flow (FMP) as a proxy for where active traders are. Cannot literally mirror a
  specific trader's book. Manage expectations: we infer crowd activity from
  volume/price/flow, not from copying named individuals.

RED (ILLEGAL — will NOT do) — hard line:
- Trading on MATERIAL NON-PUBLIC INFORMATION (actual "insider trading" in the
  criminal sense) is illegal and off the table, full stop. The GREEN items above
  use only PUBLIC disclosures (Form 4s, STOCK Act filings) — that is legal and
  is what "looking out for insider trading" will mean for us: monitoring public
  insider/congressional FILINGS, never acting on secret information.

How it fits: these change CANDIDATE GENERATION and CATALYST GRADING, not the
risk rails or entry gates. A congressionally-bought or insider-bought or
politically-catalyzed name STILL must clear: $5-9.99 band, ATR 0.04-0.25, spread
<=max($0.02,0.35%), above VWAP, RR>=1.5, $0.25 risk, resting stop, etc. They
widen the FUNNEL TOP (more/better candidates), which is exactly what a thin-tape
day like today (RVOL scanner broken, only CHPT moving) needs.
Status: PROPOSED 2026-09-04. GREEN items are low-risk to start using as research/
candidate inputs immediately (they don't touch rails). Awaiting Carlos's OK to
wire them into the daily routine + a Changelog row if he wants them formalized.

## 2026-09-04 — PROPOSAL: the real unlock — FRACTIONAL SHARES on liquid large-caps (Carlos escalation)
THE DISEASE (root cause of 7 sessions, 0 trades): the $10-max-position + whole-
shares rails force our entire universe to $5-10 stocks. That slice is structurally
junk: thin micro-caps with 0.3-5% spreads, or names pinned at the $10 ceiling we
can't follow. It is NOT gate-timidity — the tradeable universe itself is the
problem. Proof (2026-09-04 ~15:28 ET): NVDA spread $0.01 (0.004%), META $0.07
(0.011%), AMD $0.13 (0.028%, +3.4%), TSLA $0.03 (0.008%, -5.7%) vs our $5-10
names at 0.3-5%. The whole market of clean, liquid names is 10-100x better — we
just can't afford 1 whole share of a $230-616 stock on a $10 position.

THE FIX: allow FRACTIONAL shares -> trade $10 of ANY liquid large-cap (AAPL, NVDA,
META, TSLA, AMD, GOOGL, AMZN, etc.). This solves EVERY blocker at once:
- Spread gate: large-caps are penny-wide (<0.05%) — trivially passes.
- ATR ceiling: with fractional $-sizing, risk = (stop%) x $position, decoupled
  from share price; the $0.25 risk cap works on ANY price stock.
- $5-10 band + $10 ceiling: GONE — price no longer constrains the universe.
- Thin float / halt risk: GONE — mega-caps.
- Candidate scarcity / broken RVOL scanner: GONE — hundreds of liquid names,
  plus the FMP/news/congressional/insider layers now have quality names to rank.

THE TRADEOFF (honest, and why it's a RAIL change for Carlos): fractional orders
are MARKET orders, regular hours only, and CANNOT carry a resting broker stop
(platform note #4). So the broker-side stop — our core protection — is not
available on fractionals. Mitigation = MANUAL stop protocol: while in a fractional
position I monitor at 1-min cadence and fire a market SELL the instant price
trades through the stop level; hard flat by 15:50; never leave a fractional
position unattended. This converts "a monitoring gap is not a capital risk" (true
with resting stops) into "a monitoring gap COULD be a capital risk" — acceptable
only because I actively attend every position under the trend-follow tactics, and
$0.25 max risk caps the damage. THIS IS CARLOS'S CALL (risk rail).

CAPITAL MATH (must be honest): on $100 with $10/trade and $0.25 max risk, the best
realistic WIN is ~$0.20-0.50/trade. We CANNOT make "$70 in a day" on this account
- the math forbids it (a $10 position would have to +700%). Carlos's friend's $70
on META was real capital (~1+ shares = $600+). So there are TWO separate problems:
(1) we're not trading at all -> fractionals/large-caps fixes this; (2) even
trading, $10 positions make cents -> only MORE CAPITAL fixes this. Proving a
positive-expectancy EDGE on small size first, THEN scaling capital, is the whole
point of Phase 1. Recommend: approve fractionals to start generating real trades +
edge data; separately, consider funding the account beyond $100 if the $ outcomes
are to matter.

PHASE 2 note: Phase 2 = the SHORT module, gated on 20 CLOSED trades + positive
expectancy. We have 0 closed trades — skipping to it is unearned AND shorting
doesn't fix the universe problem (it adds a second way to lose on the same junk).
NOT the move now. (Though TSLA -5.7% today is exactly the kind of clean large-cap
SHORT fractionals+Phase2 would eventually open.)

RECOMMENDATION (priority order): (1) Approve FRACTIONAL large-cap trading with the
manual-stop protocol above — biggest single unlock, makes Tuesday completely
different. (2) Consider adding capital if $ results (not just edge proof) matter
now. (3) Keep shorting deferred until the long edge is proven. 
Status: PROPOSED 2026-09-04, awaiting Carlos. Rail change -> needs his dated
Changelog row before I act on it.

## Playbook — Fractional Large-Cap trading (ACTIVE 2026-09-04, Carlos-approved)

The primary path going forward. Trade small dollar amounts of GREAT instruments
instead of whole shares of junk. Retain the trend-follow tactics (early entry,
trail, quick profit, re-entry) — only the universe, sizing, and stop-mechanics change.

**Candidate universe.** Highly-liquid stocks, any price (fractional). Prefer
mega/large-caps that are MOVING today (a catalyst, sector momentum, or a clean
intraday trend) with penny-tight spreads. Sources: FMP biggest-gainers/losers +
most-active, news/political catalysts, congressional/insider disclosures (GREEN
signal layers), and the classic liquid movers (NVDA/META/TSLA/AMD/AAPL/GOOGL/AMZN/
MSFT/AVGO/NFLX/COIN/PLTR/…). The broken $5-10 RVOL scanner is no longer the
bottleneck.

**Entry gates (per trade).**
- Liquid: tight spread (<~0.05%, effectively always true for large-caps) + real ADV.
- Direction: LONG only, ABOVE VWAP (shorting still Phase-2 deferred).
- Volatility: ATR% (ATR/price) >= ~0.3% intraday — enough range for a move to clear
  friction and reach a target. (Replaces the $0.04 absolute ATR floor.)
- Trigger: a trend-follow early trigger (pullback-reclaim / micro-flag break /
  first-higher-low) or a clean breakout with room.
- Risk: define a stop level; size the fractional $-position so the loss to that stop
  is <= $0.25 max risk. Position <= $25. (Example: NVDA $230, stop 0.6% away =
  $1.38/share; to risk $0.25 buy $0.25/0.006 = ~$42 notional -> but cap at $25, so
  risk on $25 = $0.15. Fine — smaller risk is OK; NEVER exceed $0.25 or $25 notional.)
- RR: measured-move target gives >= 1.5 R.

**Execution sequence (fractional).**
1. Write ENTRY block to trade_ledger.md + commit/push (thesis, entry, stop, target, RR, size).
2. Fresh quote (<30s). Confirm spread + above VWAP + trigger still valid.
3. review_equity_order (preview) — halt/PDT check. Halt alert = ABORT.
4. Place fractional BUY (dollar-based market order, fresh ref_id UUID).
5. Read actual fill (get_equity_orders / get_equity_positions).
6. **No resting stop is possible** -> record the MANUAL stop price; go to <=1-min
   attended monitoring immediately.
7. Manage: if price trades through the stop -> fire market SELL now (fresh ref_id),
   verify flat. Else TRAIL the manual stop up (breakeven at +1R, then under higher
   lows / by ATR). Take quick profit when momentum stalls (market SELL). Re-enter
   on re-validation within the 6-order/day cap + halts.
8. HARD FLAT by 15:50 regardless (market SELL, verify zero).

**Hard safety rules (non-negotiable).**
- NEVER leave a fractional position unattended — no resting stop means a monitoring
  gap = a capital risk. If I must stop attending, flatten first.
- $0.25 max risk/trade and all loss halts (3-consec / $1 daily / $2.50 weekly) and
  the 6-order/day cap remain fully in force.
- Verify fractional order mechanics with a review_equity_order preview BEFORE the
  first real fractional entry (Tue 9/8 pre-market).

## 2026-09-04 — Strategy roadmap: quant methods (Carlos directive "think like a PhD quant")
Carlos wants economic-principle + mathematical/statistical thinking: supply/demand,
substitutes/complements, PAIRS TRADING (KO/PEP example: correlated names diverge ->
short the rich one, long the cheap one, hold until they converge), candlestick
math. Honest quant sorting — what fits our constraints NOW vs later:

PAIRS / STATISTICAL ARBITRAGE (KO/PEP, etc.) — DEFERRED (Phase 2+), honest why:
- It's a real, respected strategy (mean-reversion on cointegrated pairs, traded on
  the z-score of the spread). BUT it structurally does NOT fit us right now:
  (1) Requires SHORTING one leg — shorting is Phase-2-gated (20 closed trades +
      positive expectancy) AND can't be done on fractionals (no resting stop) AND
      needs borrow/margin we haven't verified.
  (2) It's a SWING trade — pairs diverge and reconverge over DAYS-WEEKS, not
      intraday. That collides head-on with our flat-by-close day-trading mandate.
      KO/PEP essentially never diverge+reconverge within one session.
  (3) Two legs tie up scarce capital ($100 acct) on one idea.
  (4) NOT risk-free: "until they meet again" is the classic trap — pairs can
      DE-couple permanently (earnings miss, scandal, secular shift). Convergence
      is a probability, not a guarantee; needs cointegration testing + a stop on
      the spread, not blind faith they reconverge.
  Verdict: revisit at Phase 2 (with shorting + a multi-day swing sleeve, if Carlos
  wants a separate non-day-trading bucket). Not a $100 intraday fit today.

USABLE NOW (within long-only fractional intraday rules) — the honest version of
the same economic insight:
- RELATIVE STRENGTH / rotation: among correlated names (KO vs PEP, V vs MA, NVDA
  vs AMD, XOM vs CVX), when the sector moves, go LONG the leader (momentum) or the
  mean-reverting laggard (the one lagging its pair intraday with a reclaim setup).
  Captures the "they move together" insight with a long only.
- MEAN-REVERSION math intraday: z-score of price vs VWAP, and Bollinger-band
  reversion on liquid large-caps — fade stretched moves back to the mean, or buy
  VWAP reclaims. Fits fractional large-caps + trend-follow tactics.
- VOLATILITY-NORMALIZED sizing/targets: size and set stops/targets in ATR (already
  doing via the $0.25 risk cap); rank candidates by ATR% and expected move.
- STAT DISCIPLINE: track expectancy, win rate, R-multiples, and cut by regime;
  demand statistical significance before trusting any pattern (n>=~30).

CANDLESTICK PATTERNS: Carlos referenced specific candlestick trends "he told me
about" — NOT captured in current context (pre-compaction). ACTION: ask Carlos to
restate them, or point to where he specified. Default set I'll use at S/R until
then: bullish/bearish engulfing, hammer/shooting-star, doji at support/resistance,
inside-bar breaks — as CONFIRMATION on the trigger bar, not standalone signals.

THE OVERRIDING QUANT POINT (most important, and the honest one): we have ZERO
closed trades. A real quant does NOT layer stat-arb/pairs/complex models onto a
system with n=0 — that's overfitting before a single data point. PRIORITY: get
Trade #1, build a sample of 10-20 fractional large-cap trades, measure real
expectancy, THEN add sophistication where the data says it helps. Roadmap logged;
execution stays simple until we have an edge to refine.
Status: roadmap PROPOSED 2026-09-04. Relative-strength + VWAP/Bollinger mean-
reversion are usable now within existing rules (long-only, fractional). Pairs/
short-based stat-arb DEFERRED to Phase 2. Candlesticks pending Carlos's specifics.
