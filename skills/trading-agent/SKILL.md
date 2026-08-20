---
name: "trading-agent"
description: "[v4.1.0] Execute profitable trades in your designated Agentic Account using Robinhood MCP, sizing in whole or fractional/dollar-based shares, with standing protective orders (stop-loss or take-profit) on positions, supplementary read-only market data and candidate sourcing from a connected Webull MCP, a tunable HORIZON_BIAS scale between short-term trading and long-term holding, a screen that blocks new leveraged/inverse ETF positions (e.g. TQQQ, SQQQ, SOXL), a GitHub-backed default risk parameter config whose repo, branch, and path all come from an external runtime config file (with a hardcoded fallback), and a cross-session note that carries reasoning and goals forward to tomorrow. Use it whenever the user wants to run the trading agent, execute trades, let Claude trade autonomously, start an agentic trading session, asks Claude to buy/sell stocks in the Agentic Account, or specifically mentions protective/stop-loss/take-profit orders, fractional shares, partial shares, dollar-based orders, Webull market data, horizon bias, leveraged ETF rules, risk parameters, the trading config file, or wants the agent to remember its reasoning between sessions. Also triggers on the legacy names \"V4\", \"trading skill v4\", and \"trading-skill-v4\"."
---

## Division of Responsibility

This skill defines the trading framework and **default parameters**. The invoking prompt (e.g. a scheduled session) sets timing, session-specific limits, and output format, and overrides any default here where it speaks. Where silent, GitHub-sourced or hardcoded defaults apply (see below).

## Data & execution sources

- **Robinhood MCP** is the **system of record and the only execution venue.** Every account query, position, order, protective order, and fill — and the account-scope check — goes through Robinhood, against the Agentic Account only. Nothing else ever places or mutates an order.
- **Webull MCP** (if connected) is a **read-only, supplementary market-data source** used in Step 2 for research and candidate sourcing (see "Webull market data"). It is **optional** — if it isn't connected, or a call fails, degrade gracefully and rely on Robinhood data alone; never block the session over it. **Never place orders through Webull, and never read or mix in Webull account/position/balance data** — account truth comes from Robinhood.

## Runtime configuration (load first — before Step 0)

Every environment-specific value — the GitHub token, the target repo, the file paths, the account scope — lives in **one external config file, `trading-config.json`, kept outside this repo** and provided at runtime via a connected folder. Nothing here is hardcoded to a particular user, machine, or repo.

1. **Locate `trading-config.json`** in whatever folder is connected on the current machine — never hardcode an absolute path or username. Suggested location: **`%LOCALAPPDATA%/trading-agent`** (a per-user, per-skill folder outside the repo); the same file shape is shared with `trading-report`, so a shared secrets folder works too. If no connected folder has it, request one (e.g. named `trading-agent` or `secrets`). Read it with the Read tool (plain JSON). A committed `trading-config.example.json` next to this skill shows the shape to copy.
2. **Read these values** from it and use them everywhere below in place of any literal:
   - `github.token` — PAT (`repo` scope). **Never print it in chat.**
   - `github.owner` / `github.repo` / `github.branch` — the dashboard repo coordinates. Compose the API base as `https://api.github.com/repos/{owner}/{repo}`.
   - `paths.risk_parameters` — path to the risk-parameter file in that repo (e.g. `config/risk-parameters.json`).
   - `account.scope` — the account nickname to trade in (e.g. `"Agentic Account"`). Used in Step 1 to identify the account.
3. **If the config file is unreachable** (no connected folder, no one to grant access, etc.): fall back to the hardcoded Default Risk Parameters table and the `"Agentic Account"` scope, and say so in the Step 5 report — never block the session over config.

**Notes live alongside the config.** The cross-session note `position-notes.md` (Steps 0 and 4) resides in the **same directory as `trading-config.json`** — one connected folder holds both the config and the note, so there is nothing separate to wire up. Resolve `position-notes.md` from that directory; never hardcode an absolute path or username. If the config file was unreachable and its directory can't be determined, fall back to your working/outputs directory for the same filename.

In the sections below, `$TOKEN`, `$OWNER`, `$REPO`, `$BRANCH`, and `$RISK_PATH` refer to the values read here, and `$CONFIG_DIR` refers to the directory containing `trading-config.json`.

## Loading Default Risk Parameters (do this right after loading config, before Step 0)

The "Default Risk Parameters" table is the **hardcoded fallback**. The authoritative source is `paths.risk_parameters` in the configured repo (`$OWNER/$REPO`, branch `$BRANCH`), published by the dashboard artifact when the user tunes settings. Fetch fresh every session — never reuse a prior fetch.

1. **Use the token and repo coordinates** loaded in "Runtime configuration" above. If the config file was unreachable, skip to step 4 — never block the session over this.
2. **Fetch:**
   ```bash
   curl -s -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
     "https://api.github.com/repos/$OWNER/$REPO/contents/$RISK_PATH?ref=$BRANCH"
   ```
   Decode the base64 `content` field.
3. **Validate field-by-field.** Expected keys:
   - `MAX_NEW_POSITIONS_PER_SESSION` (integer)
   - `MAX_POSITION_SIZE_PCT` (number, % of account equity)
   - `MAX_BUYING_POWER_DEPLOYED_PER_SESSION_PCT` (number, % of available buying power)
   - `CASH_RESERVE_FLOOR_PCT` (number, % of account equity)
   - `INSTRUMENTS` (`"equities_only"` or `"equities_and_options"`)
   - `DEFAULT_ORDER_TYPE` (`"limit_day"`, `"limit_gtc"`, or `"market"`)
   - `LIQUIDITY_RULE_MAX_PCT_ADV` (number — max order size as % of ADV)
   - `HORIZON_BIAS` (integer 1–5)
   - `LEVERAGED_INSTRUMENTS_BLOCKED` (boolean)

   A well-formed key replaces its hardcoded default for this session only. A missing/malformed key keeps just that hardcoded default — don't discard the whole file over one bad field.
4. **Fall back cleanly** on any failure (404, network/auth error, malformed JSON, no reachable config file): use the hardcoded table in full, and say so in the Step 5 report. Never let this block the session.
5. **Record the source** in Step 5 either way — GitHub (with fetch timestamp) or hardcoded fallback (with the reason).

This fetch is **read-only**; this skill never writes to `risk-parameters.json` (that's the dashboard artifact's job, on user request).

**Portability:** never hardcode a filesystem path or username — only check whatever folder is connected in Cowork on the current machine.

## Default Risk Parameters (hardcoded fallback)

| Parameter | Default |
|---|---|
| MAX_NEW_POSITIONS_PER_SESSION | 3 |
| MAX_POSITION_SIZE | 30% of account equity |
| MAX_BUYING_POWER_DEPLOYED_PER_SESSION | 50% of available buying power |
| CASH_RESERVE_FLOOR | 5% of account equity — never spend below this |
| INSTRUMENTS | Equities only (no options) unless the invoking prompt says otherwise |
| DEFAULT_ORDER_TYPE | Limit order, day time-in-force — the default for **whole-share** entries. Fractional/dollar-based entries take a different path (see SHARE_GRANULARITY and Step 3A). |
| SHARE_GRANULARITY | Agent's judgment — whole **or** fractional shares. Fractional/dollar-based sizing is permitted; on Robinhood it requires `type=market` + `market_hours=regular_hours` (no limit-price protection). See Step 3A. |
| LIQUIDITY_RULE | Order must be a trivial fraction of the symbol's average daily volume |
| PROTECTIVE_ORDER_POLICY | Agent judgment per position — no fixed stop/target %; set levels from the actual setup (support/resistance, volatility, thesis invalidation) and document why. At most **one** standing protective order per position at a time (Step 3B). Not GitHub-configurable by design. |
| HORIZON_BIAS | 3 on a 1–5 scale (1 = scalper, 5 = buy-and-hold). See "Horizon Bias." |
| LEVERAGED_INSTRUMENTS | Not permitted as **new** positions. See "Leveraged Instrument Screen." |

These are circuit breakers, not strategy — a single session has no memory and can't self-correct mid-flight. Precedence: **invoking prompt override → GitHub config → hardcoded table.**

## Horizon Bias

HORIZON_BIAS (1–5, default 3) is a single dial informing judgment across sizing, protective orders, turnover, and signal weighting — not a formula.

| Level | Sizing | Protective orders | Turnover | Signal weighting |
|---|---|---|---|---|
| 1 — Scalper | Smaller size, high-liquidity names only | Tight stops; take profits quickly | Days; frequent rotation fine | Momentum, relative volume, breakouts |
| 2 | Leans toward 1 | Leans tight | Days to ~1 week | Leans technical |
| 3 — Balanced (default) | Full range within caps | Judgment-based, no lean | Weeks | Mix of technical and fundamental |
| 4 | Leans toward 5 | Leans wide | Weeks to a couple months | Leans fundamental |
| 5 — Buy-and-hold | Larger, conviction sizing within cap | Wide stops off real support; reluctant to take profit early | Months; avoid churn | Fundamentals, valuation, multi-quarter trend |

Static for the session — don't drift it mid-session. If conditions argue for a different posture, say so in Step 5 rather than silently overriding; the invoking prompt or GitHub config can adjust it next session. Record each position's intended horizon in the note (Step 4) so "Watch for" can distinguish an exit-worthy dip from expected noise.

## Leveraged Instrument Screen

No **new** positions in leveraged/inverse products — 2x/3x daily-reset ETFs/ETNs such as TQQQ, SQQQ, SOXL, SOXS, UPRO, SPXU, SPXL, TNA, TZA, LABU, LABD, FAS, FAZ, UVXY, SVXY, TMF, TMV, and similar. These decay against you off a straight directional move and are structurally unsuited to holding at either end of the HORIZON_BIAS scale.

Apply before the tradability check in Step 2, per candidate:

1. **Name/description check** (primary): reject if the symbol's name/description (from `search` or `get_equity_fundamentals`) contains markers like "2x", "3x", "Ultra", "UltraPro", "Daily Bull", "Daily Bear", "Inverse", or "Leveraged".
2. **Blocklist backstop**: cross-check the list above — a floor, not exhaustive; new products launch regularly, so the name check does most of the work.
3. Log rejections under SKIPPED/REJECTED, reason "leveraged/inverse product — excluded by policy."

**New entries only** — a legacy leveraged position isn't force-closed; manage it to exit per its own thesis/protective order, noting in Step 4 that it's winding down. If unsure whether a symbol qualifies, treat it as leveraged and skip it.

## Step 0 — Read Yesterday's Note (mandatory, before anything else)

You have no memory of prior sessions — only the note your past self left.

1. Look for `position-notes.md` in `$CONFIG_DIR` — the same directory as `trading-config.json` (see "Notes live alongside the config"). If the config was unreachable, fall back to your working/outputs directory for the same filename. Check accessible directories rather than guessing.
2. If it exists, read it in full — per open position: thesis, intended horizon, plan, protective-order status and why, what would change your mind — plus a recent session log.
3. If it doesn't exist, this is either the first session or notes aren't wired up — proceed, and create it in Step 4.
4. If connected but genuinely unwritable, **say so explicitly in the report** — don't silently skip Step 4.

Treat the note as informed context, not gospel — if its thesis no longer holds given current data, say so and update it.

## Step 1 — Establish Session State (mandatory)

Rebuild live account state — the note tells you what you were thinking; this tells you what's true now:

1. Call `get_accounts`, identify the account whose nickname matches `account.scope` from the runtime config (default **Agentic Account**). Record its account number and pass it explicitly on every subsequent call — never rely on a default account.
2. Retrieve portfolio value, cash balance, available buying power.
3. Retrieve all open positions.
4. Retrieve open orders, including prior-session protective orders. For each: reconcile if filled since last session (update the note; if it was protective, note the position is now unprotected/closed); cancel any that no longer make sense at current prices, logging a reason.
5. Retrieve orders from the last 2 trading days to avoid doubling or re-entering a recent position.
6. If account data looks wrong (unexpected account, inconsistent balances), **stop and report — don't trade through anomalies.**

## Step 2 — Analysis and Decision

- Full autonomy over strategy — technical, fundamental, momentum, volatility, or combinations — informed by HORIZON_BIAS. Any symbol within the liquidity rule and instrument scope.
- Run the **Leveraged Instrument Screen** first per candidate — reject and move on before spending time on tradability or sizing.
- Managing/exiting an **existing position** is as valid a session use as opening a new one — evaluate current holdings first, informed by note and current data.
- Check `get_equity_tradability` for every remaining candidate before sizing.
- `get_earnings_calendar` can overflow a response if called broadly — narrow it (short window, relevant filters) rather than pulling the whole market.
- Document a one-line rationale per decision, including skips.
- **"No trade today" is a valid, complete outcome.** Never force a trade to have something to report.

### Saved scanners as a candidate source

Saved market scanners surface symbols matching a filter set (relative-volume breakouts, oversold RSI bounces, momentum gainers). One input among many — not a required gate.

1. Call `get_scans` — titles/filters can change session to session.
2. Call `run_scan` on scans relevant to this session's strategy. Treat results as a first pass — same leverage screen, tradability check, liquidity check, and rationale as any other candidate.
3. If a scan's filters look miscalibrated (no results, or results that don't hold up), adjust with `update_scan_filters`/`update_scan_config`, or build one with `create_scan` (use `get_scanner_filter_specs` first). Note in your rationale if you changed a scan.
4. A scan match is a reason to look closer, not a reason to trade.

### Webull market data (read-only, optional)

If a Webull MCP is connected, use it to deepen analysis and widen the candidate funnel. It is **read-only market data only** — see "Data & execution sources": no orders, no Webull account data. If it isn't connected or a call fails, skip it and proceed with Robinhood data. Reference tools by bare name.

1. **Research a candidate:** `get_stock_snapshot`, `get_analyst_rating`, `get_analyst_target_price`, `get_income_statement` / `get_balance_sheet` / `get_cash_flow`, `get_financial_indicators`, `get_stock_capital_flow`, `get_stock_industry_comparison`, `get_stock_forecast_eps`, `get_52_week_high_low`, and `get_stock_earnings_calendar`. Weight fundamental vs. technical signals per HORIZON_BIAS.
2. **Source candidates:** `get_gainers_losers`, `get_most_active`, `get_high_dividend`, and `get_market_sectors` / `get_market_sectors_detail` — a first-pass source alongside the Robinhood saved scanners above.
3. **Every Webull-sourced candidate still runs the full gate:** the Leveraged Instrument Screen, Robinhood `get_equity_tradability`, the liquidity rule, and a one-line rationale — same as any other candidate. A Webull signal is a reason to look closer, not a reason to trade.

## Step 3A — Execution Discipline (entries and exits)

Per intended trade:

1. Size within risk parameters, informed by HORIZON_BIAS. **Whole or fractional is your call** — don't round a target size down to a whole share when a fractional or dollar-based order hits the intended `MAX_POSITION_SIZE` / cash-reserve target more precisely.
2. **Choose the order path by granularity:**
   - **Whole shares →** `DEFAULT_ORDER_TYPE` (limit, day) — the default, since a marketable limit at the ask gives price protection.
   - **Fractional shares or a dollar notional →** `type=market` with `market_hours=regular_hours` (pass `quantity` with decimals, up to 6 dp, or a `dollar_amount`). This is Robinhood's **only** fractional path — a limit order can't be fractional. The tradeoff is **no limit-price protection**, so favor it on liquid, tight-spread names and check the current quote/spread first; avoid it on wide-spread or fast-moving names where a whole-share marketable limit is safer. Fractional/dollar orders are **regular-hours only** — in extended/overnight sessions, use whole-share limit orders (see `market_hours`).
3. Run `review_equity_order` (simulation) first. If it flags a problem, don't place — log rejection and reason.
4. Only after a clean review, call `place_equity_order` with the explicit Agentic Account number.
5. Record: timestamp (ET), symbol, side, quantity (note if fractional/dollar-based), order type, limit price (if any), rationale.
6. Verify fill status; note partial fills.

**Hard stops — end the session and report immediately if:**
- Cumulative deployment would exceed MAX_BUYING_POWER_DEPLOYED_PER_SESSION
- Any order call fails twice in a row (never retry a third time)
- Fills or balances don't match what you placed

## Step 3B — Protective Orders (stop-loss or take-profit)

Robinhood offers `stop_market`, `stop_limit`, and `limit` — no bracket/OCO linking a stop and target. Placing both means two independent live orders; if one fills, the other sits until a future session notices. **Place at most one standing protective order per position at a time.**

Per position held after this session (new or existing):

1. **A standing protective order generally only covers WHOLE shares** — Robinhood typically won't hold a stop against a fractional-only position (item 4 has you confirm with `review_equity_order` rather than assume). Factor this into the Step 3A sizing choice: when protection matters to the thesis (typically lower-horizon setups), prefer sizing to whole shares so the position *can* be protected; if you deliberately go fractional, accept there's no standing stop and say so in the note. For a mixed position (whole + fractional), protect only the whole-share portion. Then decide whether an order is warranted now, informed by HORIZON_BIAS — a low-horizon position more often warrants one; a high-horizon core holding you're actively watching may not. Not every position needs one every session.
2. If warranted, choose **one**:
   - **Stop-loss** (`stop_market`/`stop_limit`, sell) — downside protection. Trigger from the actual setup: below support, below thesis-invalidation, or beyond normal volatility — not a round number.
   - **Take-profit** (`limit`, sell) — locks in a target once the thesis has largely played out, or to bank gains ahead of a known catalyst (e.g. earnings). Level from resistance, a valuation target, or a risk/reward point you'd genuinely accept.
   - Never both on one position. To switch types, **cancel the existing one first**, then place the new one.
3. Use `time_in_force: gtc` — a `gfd` stop expires at the close, leaving the position unprotected overnight.
4. Fractional shares: the `place_equity_order` fractional-share note ("only on type=market... no short sells") governs fractional *buys*. Selling a fractional quantity you hold via `stop_market`/`stop_limit`/`limit` is different — `review_equity_order` tells you definitively; don't assume it can't be protected without checking.
5. Run `review_equity_order` before placing, as with any order.
6. Record in the trade log and note (Step 4): type chosen, level, and *why* — the reasoning matters more than the number, since it's what lets a future session judge whether the order still fits as conditions change.
7. No protective order is also a legitimate choice — say so explicitly rather than leaving it ambiguous whether that was a decision or an oversight.

## Step 4 — Leave a Note for Tomorrow (mandatory, before ending the session)

Write (create/overwrite) `position-notes.md` in `$CONFIG_DIR` — the same directory as `trading-config.json` (see "Notes live alongside the config"), falling back to your working/outputs directory only if the config was unreachable. This is what gives a stateless session continuity; Step 0 of the next session depends on it.

```markdown
# Position Notes — Last updated: YYYY-MM-DD HH:MM ET

## Current positions & plan

### SYMBOL — N shares @ $AVG_COST avg cost
- Thesis: why this position exists
- Horizon: intended holding horizon per HORIZON_BIAS (e.g. "3 — balanced, expect multi-week") — note "legacy leveraged holding, winding down" if it predates the Leveraged Instrument Screen
- Plan: what would make you add, hold, or exit; any known upcoming catalyst (earnings date, etc.)
- Protective order: [Stop-loss @ $X (order id) | Take-profit @ $X (order id) | None — reason]
- Watch for: the specific thing that would change your mind

(repeat per open position — remove sections for positions fully closed this session)

## Session log
### YYYY-MM-DD HH:MM ET
- What you did this session and why, in a few lines. Include protective-order changes, HORIZON_BIAS if it differed from default, any leveraged-instrument rejections, and any note updates from Step 0 reconciliation.

(keep the most recent ~10 entries; trim older ones so the file doesn't grow unbounded)
```

If `$CONFIG_DIR` (or the fallback folder) isn't writable, don't fail silently — flag it clearly in Step 5 so the user can fix the connection before the next session.

## Step 5 — Reporting

Use the invoking prompt's output format if given. Otherwise:

```
=== TRADING SESSION SUMMARY ===
Date/time: YYYY-MM-DD HH:MM ET
Account: Agentic Account (#XXXX)
Risk parameters source: [GitHub (configured risk-parameters path), fetched HH:MM ET | hardcoded fallback — reason]
HORIZON_BIAS: N (source: GitHub config | hardcoded default | overridden by invoking prompt)
Trades executed: N
TRADES: [BUY|SELL] SYMBOL x QTY @ $PRICE (status) — rationale
PROTECTIVE ORDERS: [PLACED|CANCELLED] SYMBOL — type @ $price — reason
SKIPPED/REJECTED: SYMBOL — reason (include leveraged/inverse rejections)
Account after: equity $X | cash $X | buying power $X
Note: [saved to position-notes.md | FAILED TO SAVE — see below]
Errors: none | description
===============================
```

## Important Notes

- **Agentic Account only** — verified by account number, not name, every order
- **Real money** — every order, including protective ones, is a real financial action
- **Always simulate before placing** — `review_equity_order` precedes every `place_equity_order`
- **One protective order per position, never two** — an un-triggered second order goes stale
- **Fractional = market order, whole = limit** — fractional/dollar entries are regular-hours market orders (no price protection) and can't carry a standing stop; size to whole shares when protection matters
- **Webull is read-only market data** — research and candidate sourcing only; all execution and account truth stay on Robinhood (Agentic Account)
- **No new leveraged/inverse positions** — legacy holdings wind down per their own thesis
- **HORIZON_BIAS is static per session** — informs judgment, never overrides hard risk parameters
- **Risk parameters fall back cleanly** — a GitHub fetch failure never blocks the session
- **The note reflects current reality** — update or remove stale entries, don't accumulate contradictions
