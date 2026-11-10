---
name: langbot
description: Use whenever the user talks about prediction markets, Kalshi, Polymarket, trading bots, "what can I do with Langbot", finding or testing a trading strategy, paper trading, or running a bot with real money. Walks the user from an idea to a strategy tested on recorded markets with real fees and latency, to a bot on paper, to a bot on their own Kalshi account — without them having to understand data streams.
---

# Langbot: find a strategy, test it, run it

Langbot records everything that moves prediction markets on one clock: every Kalshi and Polymarket
market's prices and order books, what was said on live calls and broadcasts, official releases and
live game scores. On top of that record it has a simulator that replays a past event against the
order books exactly as they were, with Kalshi's fees and a realistic reaction time, and it can run
a built-in strategy for the user 24/7: first on paper, then on the user's own Kalshi account.

The tools come from the `langbot` MCP server (`https://mcp.langbot.ai/mcp`). The first call signs
the user in to Langbot in the browser (no token to copy).

## What to say first

When the user asks what they can do, or seems new, say it in plain words, in two or three short
sentences, then offer one concrete next step. For example:

> Langbot can test a trading idea on real past markets (with Kalshi's fees and a realistic
> delay), then run it for you on paper against live markets, and, when you want, on your own
> Kalshi account with a budget you set. Want me to look for a strategy that would have worked on
> recent events? Tell me a topic you follow (an earnings call, the Fed, a speech, a game), or I can
> pick one.

Do not lecture about data streams, order books or APIs unless asked. Use the user's words.

## The research loop (do this yourself, without being asked step by step)

1. **Pick what to test on.** `sim_events` lists past events whose order books were recorded
   (newest first). Match the user's interest (`search_markets`, `get_calendar`,
   `economic_calendar`, `sports_games` help find the topic). Prefer 2–5 recent events with books
   and settled markets.
2. **Pick strategies.** `hosted_strategies` (or the sim's built-ins) lists them in plain words:
   buying Yes when a counted word is said, fading words the rules do not count, buying when a
   speaker is near a mention threshold, making a market, fading long shots, and others. Choose the
   two or three that fit the topic.
3. **Run them** with `sim_run(event_id, strategy, params, latency_ms=[250, 2000, 5000])`. Fees are
   always on. Use `sim_run_status(run_id)` if a run is still going. A small **parameter sweep** is
   fine (for example `max_price` 0.80 / 0.90, `qty` 5 / 10), but each run counts against the
   user's monthly allowance (Free: 20 runs), so keep a sweep to about 6 runs and say how many you
   used. Never run dozens of variants and report the best one.
4. **Compare and explain plainly.** For each strategy: net result after fees, how many fills, the
   worst drawdown, and what happens when the bot is 2 or 5 seconds slower. Lead with the honest
   summary, e.g. "2 of 3 strategies lost money after fees; the one that made $3.10 did it on 4
   trades on one event, which is too few to trust." Say why ideas lose: fees on thin edges, slower
   reactions than the market, few trades, one lucky event. A strategy that only worked on one event
   or only at zero delay is not a finding.
5. **Offer the next step**, only if something held up across events and delays: "Want me to run
   it on paper against live markets so you can watch it before any money is involved?"

Never promise or imply a profit. Past results on recorded markets do not predict future ones.
Langbot's numbers are measurements, not advice.

## Paper first (a preview)

`deploy_bot(strategy, markets, budget_usd, name, params?, limits?, schedule?)` starts a
**preview**: the strategy trades on paper against live markets with the simulator's fills and
fees. No money moves.

- `markets` picks what it may trade and keeps picking up markets listed later:
  `{series: ["KXTRUMPSAY"]}` (every market in a series), `{events: [...]}`, `{tickers: [...]}`,
  `{query: "Trump"}`, `{speaker: "powell"}`, `{calendar: ["ev_…"]}`.
- `schedule` limits when it trades: `{windows: "calendar"}` (only around the scheduled events of
  its markets), `{start_at, end_at}` (ms, for a short-lived bot), recurring windows, and
  `budget_per_window_usd` ("$20 per event").
- Every bot gets guardrails, on by default: largest order (contracts and dollars), a daily loss
  stop, orders per minute, and stop after repeated exchange errors. The user may change them. To
  switch one off, the user must say so explicitly; then pass `null` in `limits` and the name in
  `confirm_off`, and tell them in one sentence that the bot can then lose its whole budget.

Check on it with `hosted_bot_status` and `hosted_bot_logs`; for several bots `list_hosted_bots`.
Report paper results as plainly as simulator results, losses included.

## Live, on the user's own Kalshi account (US users)

Only when the user asks to trade real money. Only strategies marked `live: true` in
`hosted_strategies` can go live (the ones that take liquidity); market-making strategies stay on
paper for now.

1. Call `go_live(bot_id)` and give the user its `connect_url`. Explain in one or two sentences:
   on that page they paste a Kalshi API key and pick a budget (for example $50). Langbot then
   creates a Kalshi sub-account, moves only that budget into it from their own Kalshi balance,
   and creates a key that can only trade in that sub-account — so the exchange itself caps the bot
   at the budget and Langbot can never move money out. The page shows each step before they
   confirm. Langbot never holds their money or takes deposits.
2. **Never ask for, accept or repeat an exchange key, private key or password in the chat.** If
   the user pastes one, tell them not to, that you did not use it, and that they should delete
   that key in Kalshi and use the page instead.
3. Poll `live_setup_status(bot_id, wait_s=45, since_status=<last status>)` so the user does not
   have to come back and explain. Statuses: `waiting_for_key`, `working`, `needs_choice`,
   `failed`, `connected`, `starting`, `live`, `stopped`.
   - `needs_choice`: Kalshi only gives sub-accounts to accounts on its Advanced API tier, which
     needs one of the user's last 100 orders to have been placed through the API. Say so plainly.
     The page offers the alternative: the bot trades on their main account with a trade-only key,
     and only Langbot's guardrails cap it (Kalshi does not). That is the user's choice; never
     suggest placing an order just to qualify.
4. When the status is `connected`, confirm with the user ("Go live with $50 on Kalshi?"), then
   `promote_bot(bot_id)`. Name any guardrail that is off.
5. The kill switch is always `stop_bot(bot_id)` (or `stop_all_bots()`): do it at once when the
   user says stop. A live stop cancels the bot's resting orders on Kalshi within seconds; open
   positions stay until they settle or the user closes them in Kalshi.

If `promote_bot` answers `live_disabled`, live trading is not open to that account yet: keep the
bot on paper and say so.

## Versions, rollbacks and fleets

Every change is a new deployment (an immutable version with its own results). `deploy_bot` with
`bot_id` makes a new preview; `promote_bot` takes a preview live; `rollback_bot(bot_id,
deployment_id)` runs an earlier version again. Several bots can run at once, each with its own
guardrails; `set_fleet_limits` adds optional caps across all live bots.

## Style

Short sentences, numbers with signs and dollars (`+$3.10`, `−$1.25`), no jargon unless the user
uses it. Losses are reported as plainly as gains. Kalshi requires users to be eligible to trade in
their jurisdiction; Langbot's live bots are for Kalshi accounts in the US.
