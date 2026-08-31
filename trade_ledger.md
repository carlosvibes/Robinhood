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
| 2026-08-28 | $100.00 | $100.00 | $0.00 | 0.0% | 0 | First full session from pre-market. 3 setups tracked (WEN ORB/flag, DFDV flag, VISN reclaim) — none confirmed; all refusals validated by outcome. Flat verified 15:45 ET. |

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

10:02 ET — WEN at the trigger but NOT confirmed: 9:55 bar closed exactly AT the OR
high 8.07 (not above), vol 194k vs ~206k OR-bar avg (just under). Now 8.085, VWAP
8.005 (above ✓). Waiting for the 10:00 bar close (10:05): needs close > 8.07 on
vol > 206k, then entry on a RETEST at/below 8.07 — no chasing. Pre-computed plan
if confirmed: entry ~8.05-8.06, stop 7.96 (OR mid), target 8.29 (H+R), RR ~2.3,
1 share (~$8.06 position, $0.10-0.11 risk). ATR check pending. Others: VISN fading
6.635 no reclaim; DFDV bouncing 5.23 but below OR mid; FWDI 6.625 below OR high.

10:27 ET — WEN RESOLVED WITHOUT ENTRY. Sequence: 10:05 close 8.085 (7k vol, trivial);
10:10 dip to 8.03 held VWAP; 10:15 closed 8.125 on 110.7k (< 206k OR-bar avg →
never met the volume confirmation); now 8.125, high 8.158. Entry at current price
fails geometry (risk 0.165 / reward 0.165 = 1.0:1) — chase banned. Rule conflict
logged for learning.md: OR-bar average volume (~206k) may be an unreachable
confirmation bar after 10:00 because opening bars carry peak volume; the breakout
"worked" without ever confirming. Candidate proposal: compare breakout volume to
the average of the prior N non-OR bars instead. NOT ACTIVE — needs Carlos approval.
Watching for a pullback that holds 8.07-8.08 (entry there = 2.0-2.3:1, still valid).

10:42 ET — WEN: THE RETEST CAME AND WENT BETWEEN CHECKS. 10:25 bar low was 8.07
exactly (the level), held, closed 8.095; now 8.185 with high 8.19. That was the
playbook entry (~8.08, would now be +$0.10 with target 8.29 live). Missed on
monitoring cadence: retest window ~4 min, check interval 15 min, and observed
wake latency has been up to 20 min. Entry now fails 2:1 from every stop (0.5-0.9:1)
— letting it go per rule, not chasing.
LESSON (operational, adopting immediately — cadence is not a risk rail): while a
setup is ARMED near its trigger, re-arm checks at ~5 min, not 15. Also log for
learning.md: wake delivery latency (up to +20 min observed) means time-critical
entries need the tightest cadence the scheduler allows.
VISN bouncing 6.41→6.52 on rising vol (331k bar) — possible VWAP reclaim forming,
not confirmed. DFDV 5.37 (+1.7%) recovering. FWDI 6.53 red. Next check 5 min.

10:50 ET — TWO SETUPS ARMING. WEN bull flag: impulse 7.85→8.19 (0.34), shallow flag
8.11-8.19 on declining vol (24% retrace, clean per playbook 2). Trigger: 5-min close
over 8.19 on expanding vol. Pre-plan: entry ~8.19-8.20, stop 8.11 (below flag low,
risk ~0.09), target 8.53 (impulse from flag high) = ~3.7:1. ATR + spread + bid-size
checks at entry time. DFDV: broke morning range to 5.475 on 113k (biggest bar since
open), testing yesterday's high 5.51 — entry HERE fails RR (~0.4:1 to measured
target); valid only if a flag forms above ~5.40 (impulse 5.03→5.51 = 0.48 projects
~5.98 on a flag break). VISN 6.515 still below VWAP 6.583 — dormant. Cadence 5 min.

10:57 ET — Flags intact. DFDV now primary: flag 5.43-5.54, declining vol (43k→25k),
22% retrace of 0.51 impulse. Trigger = 5-min close > 5.54 on expanding vol; entry
~5.55, stop 5.42, target ~6.05 (impulse from flag high), ~3.9:1, 1 share. WEN flag
8.15-8.19 holding BUT ATR compressed to 0.047 < $0.05 floor — currently fails the
universe ATR check; re-verify at any trigger (a breakout bar may restore it).
VISN 6.525 below VWAP 6.58 — dormant. Cadence 5 min.
11:02 ET — 10:55 bar: DFDV touched 5.54 (trigger) but closed 5.51 — not through, vol
expanded 65k. Flag valid, higher low 5.46. WEN 8.15 flag bottom. VISN 6.535 nearing
VWAP 6.58 on 171k. No triggers. Cadence 5 min.
11:18 ET — DFDV: THIRD closing rejection of 5.54 (11:05 high 5.57, close 5.50; 11:10
close 5.51). Setup degrading — level actively defended. WEN sagging to 8.12, one tick
above flag floor 8.11. Midday window starts 11:30 (default no new entries); both
setups expire then unless triggered. VISN dead. Last pre-midday check ~11:25.
11:28 ET — No trigger by midday cutoff. DFDV never closed above 5.54 (4 tests, vol
now dried to 1k/bar); WEN never closed above 8.19. Both morning setups EXPIRED.
Midday mode: no new entries 11:30-14:00, 30-min checks (RVOL scan + structure watch).
Morning summary: 0 trades, 0 orders, $100.00 intact, 3 setups tracked, all correctly
refused or unconfirmed. Afternoon hunting resumes 14:00.
12:31 ET midday check — RVOL scan: 0 names (quiet Friday). Morning refusals all
validated by outcome: DFDV broke DOWN to 5.12 (refused entry at 5.55 would have
stopped at 5.42 = -1R avoided), VISN sank to 6.13 (failed reclaim read correct),
WEN 8.175 never confirmed. Refusal scorecard today: 3/3 would-have-been losers
avoided. Still flat, $100.00. Next check 13:30; entries resume 14:00.
13:32 ET — RVOL scan: 1 name (VISN, RVOL 2.27, -47%, still no long structure). WEN
coiled 8.16-8.20 for 45+ min pressing the high — afternoon breakout candidate:
trigger close > 8.20 on volume expansion, stop 8.11 (flag low), target 8.54
(0.34 impulse from 8.20), ~3.3:1. CAUTION — two rails to verify at entry: ATR
(was 0.047 < 0.05 floor and compressing) and RVOL (WEN dropped OUT of the RVOL
scan → may read < 2.0 now). Both are hard gates. Entries resume 14:00.
14:02 ET — Afternoon window open, ZERO qualifying candidates. WEN ELIMINATED: ATR
collapsed to 0.024 (< 0.05 floor, hard universe fail) + RVOL < 2.0 (out of scan).
Note: its only close above 8.20 came at 13:30 (inside no-entry window) on weak
volume and faded back — the trigger discipline validated yet again. VISN: only
scanner name, -47%, no long structure all session. Expect a 0-trade close to
week 1; weekly review at 15:45 will compile the funnel data for the RVOL-threshold
discussion with Carlos. Checks: 14:30, 15:00; flatten 15:45.
14:31 ET — Scan: VISN only (still -47%, 6.11, downtrend intact — no long structure).
No new names, WEN/DFDV unchanged in their disqualified states. No candidates.
15:01 ET — Final entry check: VISN only (unchanged, -47%, no long structure). No
entries this week. Entry window closed; flatten + EOD + weekly review at 15:45.

### Session — 2026-08-28

Session hours (from marketHours):  open 09:30 — close 16:00 ET
Flatten time:                      15:50 ET (close - 10 min)
Blocking checks:                   STATUS ok / broker reconciled / buying power $100.00
                                   / clock verified (08:35 ET)

Pre-market watchlist:  DFDV (SOL-treasury catalyst confirmed via news), VISN (-41%
                       overnight crash into band), FWDI (multi-day momentum), WEN
                       (post-crash stabilization — became the day's main candidate)
Market context:        SPY flat, Nasdaq futures -0.3% pre-market. No index tailwind.

Opening orders submitted:  0  (cap 6)
Filled / Rejected:         0 / 0
Winners / Losers:          0 W / 0 L
Consecutive loss streak at close:  0  (halt at 3)
Gross P&L:             $0.00
Net P&L:               $0.00  (0.0% of start-of-day equity)
Ending equity:         $100.00
Position count at flatten (verified by query at 15:45 ET):  0

Halt conditions hit:   none
Incidents:             none
Rule conflicts:        WEN ORB — two closes AT 8.07 (never above) on sub-threshold
                       volume, then the clean 10:25 retest was missed between checks
                       (cadence, not rules); WEN's 13:30 close above 8.20 came on weak
                       volume inside the midday window and faded (trigger discipline
                       validated). DFDV — four closing rejections of 5.54, then broke
                       DOWN to 5.12 (refusals validated: entry would have been -1R).
                       VISN — reclaim never closed above VWAP; kept bleeding to 6.11.
                       WEN eliminated at 14:02 on ATR floor (0.024 < 0.05) + RVOL < 2.

Best decision today:   Not chasing WEN at 8.125/8.185 after the missed retest — both
                       chase geometries computed below 1:1 and were refused.
Worst decision today:  15-min check cadence during an armed setup cost the one clean
                       entry of the day (WEN 10:25 retest, ~4-min window). Fixed
                       intraday: 5-min cadence while armed, adopted from 10:42 on.
Platform/data issues:  Wake delivery latency up to +20 min observed on scheduled
                       checks — noted in learning.md; broker-side resting stops mean
                       open positions never depend on wake timing for protection.

### Session in progress — 2026-08-31 (Monday, week 2)

Blocking checks (08:32 ET): STATUS ok / broker flat (0 pos, 0 orders) / buying power
$100.00 / clock verified. Weekly loss counter reset. learning.md week-1 review read.

Market context: SPY -0.3% pre (766.88 vs 769.35). No fresh gappers in band — gap
scan returned only VISN and that gap is Friday's candle (known filter behavior:
1d gapRatio reads the last completed daily candle pre-market).

Pre-market watchlist (08:33 ET):
1. FWDI $6.11 (+3.7% pre) — only pre-market mover; multi-day momentum name. Needs
   ATR >= 0.05 and RVOL > 2 at any trigger.
2. WEN $8.20 (-0.8% pre) — held Friday's gains (closed 8.27). Continuation over
   Friday high 8.29 only with returned volume + ATR/RVOL gates.
3. VISN $6.07 (flat) — post-crash bounce watch, VWAP-reclaim only, no fresh news.
OUT: DFDV $4.99 — below the $5.00 price floor, out of universe.

Plan: opening ranges 9:30-9:45, then playbook hunting per week-1 process.

09:48 ET — Opening ranges (9:30-9:45):
  VISN: H 6.36 / L 6.19 / R 0.17 / M 6.275 — now 6.25, VWAP 6.289 (just below).
        PRIMARY ARMED SETUP: ATR 0.057 >= 0.05 ✓, massive volume (864k first bar,
        RVOL trending >2), R 2.7% of price ✓. Trigger: 5-min close > 6.36 on vol >
        ~375k OR-bar avg (break also clears VWAP). Plan: retest entry <= 6.36,
        stop 6.275 (M), target 6.53 (H+R), risk ~0.085 (>= 1x ATR ✓), RR 2.0,
        1 share. Day-2 bounce on Friday's -47% crash = genuine volatility.
  WEN:  H 8.38 / L 8.22 / R 0.16 / M 8.30 — 8.32, above VWAP 8.298. Structure ok
        BUT ATR 0.042 < 0.05 (rising as Friday's coil rolls off — recheck at any
        trigger) and today's volume is light (RVOL likely < 2). Secondary.
  FWDI: H 5.96 / L 5.83 — weak, red-ish, light volume. Shelved.
DFDV still < $5.00, out. Cadence: 5 min (VISN armed).
09:54 ET — 9:45 bar: VISN 6.27 (inside OR, below trigger 6.36, below VWAP — armed,
intact). WEN 8.275 below OR mid, weakening. No triggers. Cadence 5 min.
10:01 ET — VISN 9:50 bar tagged 6.36 exactly (close 6.31, vol 236k rising, higher
low at OR mid 6.275); 9:55 bar tight pause 6.29-6.315 on fading vol — constructive
coil under the trigger, near VWAP. Still needs the closing break > 6.36. Armed.
WEN 8.28 stagnant. Cadence 5 min.
10:11 ET — VISN 10:05 bar CLOSED 6.365 (above trigger 6.36, high 6.40) on 177k —
volume confirmation FAILED by the rule as written (requires > OR-bar avg 375k,
which is inflated by the 864k opening bar). Second documented occurrence of the
OR-volume-bar structural issue (WEN Friday was the first — that breakout worked
without ever "confirming"). Rule applied strictly: NO ENTRY. Proposal evidence
strengthened for Wednesday's review. Setup stays armed: a close > 6.36 on >= 375k
confirms properly; close < 6.19 kills it. Cadence 5 min.
10:20 ET — VISN 10:10 bar: second consecutive close above trigger (6.37, held
6.35-6.37) on 40k — still no 375k confirmation bar. Price extending away without
ever "confirming" under the current rule, mirroring WEN Friday. Logged as further
proposal evidence. Setup armed (needs close > 6.36 on >= 375k; dies < 6.19).
Volume-rule proposal is in learning.md AWAITING Carlos's changelog approval.
