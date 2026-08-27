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
