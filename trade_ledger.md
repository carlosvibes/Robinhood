STATUS: ACTIVE

<!--
The line above is machine-checkable state and MUST be the first line of this file.
Valid values:
  ACTIVE
  HALTED_DAY <YYYY-MM-DD>
  HALTED_WEEK <YYYY-MM-DD> — awaiting approval

The agent reads this before anything else each session and must not trade when halted.
Only Carlos clears HALTED_WEEK, by editing this line himself.
-->

# Trade Ledger

Permanent trading database. Everything below the STATUS line is **append-only.** Never delete or edit a past entry — if something was recorded wrong, append a correction with the timestamp of the correction. The value of this file comes entirely from it being an honest record, including the parts that are unflattering.

All times are US Eastern. All dollar figures to two decimals.

---

## Account log

| Date | Starting equity | Ending equity | Day P&L | Day P&L % | Trades | Notes |
|---|---|---|---|---|---|---|
| 2026-08-26 | $100.00 | | | | | Account funded with $100.00 (deposit pending as of 2026-08-26 evening; buying power already $100.00). Phase 1. No trades yet. |
| 2026-08-27 | $100.00 | $100.00 | $0.00 | 0.0% | 0 | First live session (started mid-day 12:05 ET). No setup cleared all rails. Flat verified at 15:46 ET. |

---

## Trade entry schema

Every trade gets one block. The **Entry** section is written to disk *before* the confirming quote is fetched and before the order is placed — that ordering is what makes the drift check meaningful rather than circular. The **Fill** and **Exit** sections are written from what the broker actually reports, never from what was intended.

```
### Trade #[N] — [TICKER] — [YYYY-MM-DD]

**Status**: OPEN | CLOSED | INCIDENT

--- ENTRY PLAN (written before the confirming quote, before the order) ---
Time planned:         HH:MM:SS ET
Strategy:             ORB | Momentum | Breakout | VWAP Reclaim
                      (MA Cross is confirmation only and is not a valid strategy value)
Phase:                1 | 2 | 3
Analysis price:       $
Setup trigger level:  $      [the ORB high / consolidation high the thesis hinges on]
Planned entry:        $
Planned shares:       [whole shares unless platform notes confirm fractional stops work]
Position value:       $
Stop price:           $
Target price:         $
Risk ($):             $
Reward:risk:          X.XX : 1     [must be >= 2.00 — refuse the trade if not]
% of equity at risk:  X.XX%

Universe filter:
  Price $5.00-[phase cap]:  yes/no  (actual: $)
  Avg daily volume >500k:   yes/no  (actual: )
  Relative volume >2.0:     yes/no  (actual: )
    Source:                 scanner | manual fallback
      Manual fallback = cumulative vol since open ÷ median cumulative vol at
      the same clock time over the prior 20 sessions.
      Never write a number you did not compute.
  Spread within limit:      yes/no  (actual: $     =    % of price)
      Limit is the LARGER of $0.01 or 0.15% of price. Below $6.67 a single
      tick already exceeds 0.15%, so the penny allowance is what keeps the
      $5.00-$6.67 band tradeable.
  Float >10M shares:        yes/no  (actual: )
  ATR(14) on 5min bars:     $        [must be >=$0.05]
  Stop distance / ATR:      X.XX     [must be >=1.00 — a tighter stop is
                                      inside normal noise]
  Bid size >=10x my shares: yes/no  (actual: )
  Exchange:                 NASDAQ/NYSE
  Not halted / no SSR:      yes/no

VWAP context (checked on every trade — long setups require price above VWAP):
  VWAP at entry:            $
  Price above VWAP:         yes/no
  Times price crossed VWAP today:      [3+ means the level is not holding]

review_equity_order alerts:
  Called at:                HH:MM:SS ET
  Buying power alert:       none / [detail]
  PDT alert:                none / [detail — if present, log to platform notes and escalate]
  Halt alert:               none / [ANY halt alert is an absolute stop]
  Estimated cost:           $

Verification sequence:
  Entry plan written to disk at:  HH:MM:SS ET
  Live quote fetched at:          HH:MM:SS ET
  Quote timestamp:                HH:MM:SS ET   (must be within 30s, regular hours)
  Live price:                     $
  Drift vs analysis price:        X.XX%   (must be <1%)
  Drift vs trigger level:         X.XX%   (must be <0.5%, and for ORB the entry
                                           must be at or BELOW the trigger level)
  Symbol resolved:                yes/no
  RR recomputed from live price:  X.XX : 1

Catalyst:             [news / earnings / none — with source and timestamp]
Context signals:      [insider or congressional data if any — context only, never the thesis]

**Entry thesis** (2-4 sentences, written before the order):
Why this setup, why now, what specifically confirms it, and what would prove it wrong.

**What would invalidate this**:
The specific observable thing that means the idea is dead, beyond the stop being hit.

--- FILL (read back from the broker — never assumed) ---
Order type:           marketable limit
ref_id (idempotency): [UUID generated BEFORE placing. On a transport failure,
                       retry once with this SAME id. Any other failure: stop
                       and reconcile against the broker.]
Limit price:          $
Time filled:          HH:MM:SS ET
Filled shares:        [actual, may differ from planned]
Average fill price:   $
Slippage vs quote:    $        (   %)   [escalate if >0.3%]
Partial fill:         yes/no   [if yes, ALL downstream math uses filled quantity]

Resting stop order:
  Placed at:          HH:MM:SS ET
  Order type:         stop-market
  ref_id:             [its OWN fresh UUID — not the entry's]
  Order ID:           
  Accepted:           yes/no
  [Explicitly REJECTED  -> close the position immediately. Log as INCIDENT.
   Timed out / ambiguous -> call get_equity_orders FIRST to find out whether
   the stop actually rested. Do not close blind: if the stop did rest, it holds
   the shares and the close fails on insufficient quantity, leaving you
   believing you hold something you can neither protect nor exit.]

--- EXIT ---
Stop cancelled first:  yes/no — cancellation VERIFIED by querying open orders?
                       [No OCO brackets exist, so a resting stop holds the
                        shares. Selling without cancelling it first fails.]
Exit order ref_id:     [its OWN fresh UUID. On a flatten, a retry without one
                        can double-sell and open a SHORT — this is a margin-
                        capable account, so it will actually execute.]
Time out:             HH:MM:SS ET
Exit price:           $
Exit shares:          [must equal filled shares — verify position is zero by query]
Exit reason:          Target | Stop | Flatten | Invalidated | MA cross-down | Halt | Incident
Hold duration:        
Gross P&L:            $
Spread/slippage cost: $
Net P&L:              $
Net P&L %:            
R multiple:           [net P&L ÷ planned risk — e.g. +2.1R, -1.0R]

**What actually happened**:
Plain account of the price action versus what was expected.

**Execution quality**:
Did the entry fill where intended? Was the stop respected? Was the exit taken at plan,
early, or late? Grade the *decision* separately from the *outcome* — a losing trade
executed exactly to plan is a good trade, and a winning trade taken off-plan is a warning.
```

---

## Incident schema

For anything that went wrong mechanically rather than directionally. These matter more than losses — a loss is the system working, an incident is the system failing.

```
### INCIDENT — [YYYY-MM-DD HH:MM:SS ET] — [TICKER if applicable]

Type:         Ambiguous order response | Rejected stop | Partial fill | Trading halt |
              Ledger/broker mismatch | Tool error | Forced overnight hold | Other
What happened:
Broker state at time of incident:   [positions and open orders as actually queried]
Action taken:
Resolved:     yes/no
Escalated:    yes/no — at HH:MM
Follow-up needed:
```

---

## Daily session summary schema

```
### Session — [YYYY-MM-DD]

Session hours (from marketHours):  open HH:MM — close HH:MM ET
Flatten time:                      HH:MM ET (close - 10 min)
Blocking checks:                   STATUS ok / broker reconciled / buying power >$0
                                   / clock verified

Pre-market watchlist:  [tickers, one line each on why]
Market context:        [SPY direction, general volatility, anything unusual]

Opening orders submitted:  N  (cap 6)
Filled / Rejected:         /
Winners / Losers:          W / L
Consecutive loss streak at close:  N  (halt at 3)
Gross P&L:             $
Net P&L:               $  ( % of start-of-day equity)
Ending equity:         $
Position count at flatten (verified by query):  0

Halt conditions hit:   none | [which, at what time, at what equity]
Incidents:             none | [reference the incident blocks above]
Rule conflicts:        [setups passed on because a rule blocked them — worth tracking,
                        since a rule that blocks many good trades may need review]

Best decision today:
Worst decision today:
Platform/data issues:
```

---

## Trades

*(Entries begin below. Newest at the bottom — chronological order makes the progression readable.)*

### Session in progress — 2026-08-27 (first live session; started mid-day ~12:05pm ET)

Session hours: 09:30–16:00 ET (normal day) | Flatten: 15:50 ET
Blocking checks: STATUS ok / broker reconciled (flat, 0 orders) / buying power $100.00 / clock verified

Watchlist at 12:05pm ET (from saved scan "Phase 1 Day Trader Universe"):
- XHLD (TEN Holdings) $9.48, +11.7%, RVOL 3.66, float 10.3M — REJECTED at spread check:
  bid/ask 9.40/9.59, spread $0.19 ≈ 2.0% of price vs $0.014 limit. Not tradeable.

Tradeable candidates: 0. Mid-day window (no new entries before 14:00 by rule).
Plan: re-scan at ~13:55 ET for the afternoon window; entries only on a genuine setup
passing every rail; flatten verification at 15:45–15:50 ET; full summary at EOD.

13:56 ET re-scan: 2 names. WEN $7.79 (spread ok at $0.01, but -13.8% day and below
VWAP 7.88 → no long; watch only for a VWAP reclaim on volume). XHLD $8.47 (spread
$0.14 ≈ 1.7% → fails again; also below VWAP 8.62). Tradeable candidates: 0.

14:35 ET check: 4 scanner names. DFDV (+17%, $5.27) PASSES universe filter — spread
$0.01 at limit, above VWAP 5.248, ATR $0.054, float 22.6M — but NO valid setup:
rejected at HOD 5.51 (13:50), lower high 5.495 (14:05), then heavy-volume selldown
to 5.24 tagging VWAP; pullback retraced ~77% of the prior leg = reversal, not a
flag per playbook. No catalyst verifiable (FMP news tier-blocked). Watching for a
base above VWAP + volume break. WEN still below VWAP (7.805 vs 7.875) — no reclaim.
XHLD spread $0.09 fails. NMZ: flat muni CEF, no setup. Trades so far: 0.

15:10 ET final entry check: 6 scanner names. DFDV based above VWAP 5.23-5.35 but
geometry fails — breakout entry ~5.36, stop below base 5.23 = $0.13 risk vs $0.15
reward to HOD 5.51 = 1.15:1 < 2.00 gate; VWAP also crossed 3+ times (level not
holding). FWDI $6.635 below VWAP 6.748 AND ATR $0.035 < $0.05 floor. PLAY -7.1%
below VWAP. WEN no reclaim. XHLD spread. NMZ no setup. Entry window closed.
DAY RESULT: 0 trades — no setup cleared all rails. Flatten check at 15:45.

### Session — 2026-08-27

Session hours (from marketHours):  open 09:30 — close 16:00 ET
Flatten time:                      15:50 ET (close - 10 min)
Blocking checks:                   STATUS ok / broker reconciled / buying power $100.00
                                   / clock verified

Pre-market watchlist:  none — session started mid-day at 12:05 ET (first live session,
                       launched on Carlos's go-ahead after the market-hours verification)
Market context:        SPY ~flat vs prior close. Scanner names were mostly single-stock
                       stories: WEN -13.7% selloff day, DFDV +18% crypto-treasury mover,
                       XHLD faded gapper, FWDI +12% fader.

Opening orders submitted:  0  (cap 6)
Filled / Rejected:         0 / 0
Winners / Losers:          0 W / 0 L
Consecutive loss streak at close:  0  (halt at 3)
Gross P&L:             $0.00
Net P&L:               $0.00  (0.0% of start-of-day equity)
Ending equity:         $100.00
Position count at flatten (verified by query at 15:46 ET):  0

Halt conditions hit:   none
Incidents:             none
Rule conflicts:        Candidates rejected, by rail: spread gate (XHLD twice, ~1.7-2.0%
                       of price vs 0.15% limit); VWAP direction filter (WEN all afternoon,
                       FWDI, PLAY); 2:1 RR gate (DFDV — best structure gave 1.15:1);
                       ATR >= $0.05 floor (FWDI). None of these look like rules blocking
                       good trades; each rejection had a clear cost-based reason.

Best decision today:   Passing DFDV at 14:35 — it looked buyable (above VWAP, spread ok)
                       but the 77% retrace said reversal; it then chopped sideways-down
                       and never paid. The RR gate refusal at 15:10 was also correct.
Worst decision today:  None material. Session design note: a mid-day start offers
                       structurally fewer setups — ORB impossible, prime window missed.
Platform/data issues:  FMP news endpoint is tier-blocked, so catalysts cannot be
                       verified — logged to learning.md as an open limitation.

### Session in progress — 2026-08-28 (second live session; full day from pre-market)

Session hours: 09:30–16:00 ET (normal day) | Flatten: 15:50 ET
Blocking checks (08:35 ET): STATUS ok / broker flat (0 pos, 0 orders) / buying power
$100.00 / clock verified. learning.md read.

Market context: SPY flat pre-market (771.50 vs 771.10 close), Nasdaq futures -0.3%.

NEW CAPABILITY: Robinhood MCP now exposes get_equity_news — catalyst verification
works. Yesterday's "unverifiable" limitation is resolved. (Also: main RVOL scan
returns 0 pre-market as predicted in learning.md; built a saved gap scan instead —
"Pre-market Gappers $5-10", scan_id 4552457e-8b53-40c5-9f81-f987eb23252f.)

Pre-market watchlist (08:36 ET):
1. DFDV $5.33 (+0.9% pre) — CATALYST CONFIRMED (MT Newswires 8/27 14:17): resumed
   SOL purchases, ~19k SOL, treasury ~2.3M SOL. Yesterday +18% on the news. Play:
   momentum continuation / breakout over yesterday's high 5.51 if volume confirms.
2. VISN $6.78 (-41.5% overnight, from 11.63) — crash into our band; catalyst
   unclear from news (only a buyback note). Float 223M, avg vol 8M. Long-only =
   VWAP-reclaim ONLY, and only on a closing 5-min candle above VWAP w/ volume.
   Crash-day chop expected; extreme caution, likely no trade.
3. FWDI $6.90 (+3.6% pre) — multi-day momentum (news 8/20-8/24: repeated 8-12%
   up moves); faded yesterday afternoon, ATR was thin ($0.035). Needs ATR >= $0.05
   and a clean flag to qualify.
4. WEN $7.84 (flat) — post-crash stabilization, no catalyst for a long. Low priority.

Plan: 9:30-9:45 record opening ranges, NO entries. 9:45+ hunt ORB retest per
playbook geometry (R <= $0.50, R <= 6% of price, entry at/below range high, 3e >= 2s).

09:47 ET — Opening ranges (9:30-9:45):
  WEN:  H 8.07 / L 7.85 / R 0.22 / M 7.96 — now 7.97 (+2.0%), strongest name.
        ORB watch: needs 5-min close > 8.07 on vol > OR avg (~206k/bar). Entry on
        retest at/below 8.07, stop 7.96, target 8.29 (H+R). Geometry: risk 0.11,
        reward 0.22 = 2.0:1 at trigger exactly — prefer entry a cent or two below.
  DFDV: H 5.40 / L 5.08 / R 0.32 / M 5.24 — now 5.06, BROKE below OR low. Red on
        day, below yesterday close. Continuation thesis dead; no long structure.
  FWDI: H 6.73 / L 6.41 / R 0.32 / M 6.57 — now 6.48, below mid, fading.
  VISN: H 6.85 / L 6.53 / R 0.32 / M 6.69 — now 6.71, mid-range crash-day chop.
        Massive volume (890k first 15 min). VWAP-reclaim only, none formed.

FINDING (answers learning.md open question #1): RVOL scan returned 0 at 09:46 even
with VISN at ~890k shares in 15 min — FILTER_TYPE_RELATIVE_VOLUME at 1d does NOT
normalize for time of day. Morning scans are structurally empty; use the gap scan
(4552457e) + watchlist in the morning, RVOL scan from ~midday on.

No entries. Next check 10:00 ET.
