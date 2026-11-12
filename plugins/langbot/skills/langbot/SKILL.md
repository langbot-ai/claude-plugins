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

Do not lecture about data streams, order books or APIs unless asked. Use the user's words. The
`langbot-start` prompt (`langbot://start`) has the three steps for a new user: pick a bot and
run it on paper, watch it, go live with a Kalshi key given here. Drive that loop yourself: after
each step, offer the next one.

## The short path: core tools

The tools whose description starts with `[core]` are the short path. Use them first and in this
order; reach for the rest only when the user asks something they do not answer.

1. Find markets: `search_markets`.
2. Test on recorded books: `sim_events`, `sim_run` (`sim_run_status` while a run is going).
3. Check and deploy on paper: `hosted_strategies`, `preflight_bot`, `deploy_bot`.
4. Watch it: `bot_report`, `hosted_bot_trades` (its orders and fills, in dollars).
5. Go live, after the user says yes: `connect_kalshi`, `promote_bot(bot_id, confirm=true)`.
6. Stop: `stop_bot`, or `kill_switch` for every bot at once.

The rest, by group: finding a topic (`list_live_events`, `get_calendar`, `economic_calendar`,
`official_releases`, `sports_games`, `game_timeline`, `get_market_history`, `market_brief`);
history and models (`get_base_rates`, `search_outcomes`, `search_mentions`, `list_speakers`,
`get_calibration`, `get_fair_value`, `get_live_model`, `get_consolidated`); more on a hosted
bot (`list_hosted_bots`, `hosted_bot_status`, `hosted_bot_logs`, `update_bot`, `rollback_bot`,
`live_setup_status`, `go_live`, `stop_all_bots`, `set_fleet_limits`); receipts (`get_receipt`,
`verify_receipts`, `export_replay`, `attest`).

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

## Paper first

Before deploying, call `preflight_bot(strategy, markets, budget_usd, params?)`. It says how many
open markets the selection matches now, how many pass the strategy's own filters now and over the
last 7 days, the parameters fit to the budget and to those markets (`effective_params`, each with
its source: `you`, `budget`, `markets`, `default`), and `expected.trades_per_day` on last week's
recorded books. If `status` is `blocked`, read `problems` to the user in plain words and apply a
`suggestions` entry (a parameter and a value, or other markets) before deploying, e.g. "these
markets close within a day; I'll set min_hours_to_close to 6". Only pass `force=true` when the
user wants it deployed anyway.

`deploy_bot(strategy, markets, budget_usd, name, params?, limits?, schedule?)` starts the bot
**on paper** (the API names this target `preview`; to the user say "on paper", and "go live"
rather than "promote", the words langbot.ai uses): the strategy trades on paper against live markets with the simulator's fills and
fees. No money moves. Leave out parameters the user did not choose: money and contract sizes are
fit to the budget (a $5 bot gets $5 limits and 1-contract orders) and time-to-close filters to the
markets. Tell the user the effective values that matter (`deployment.effective_params`).

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

Check on it without being asked twice: `bot_report(bot_id)` is a short plain-text summary to show
the user (no `bot_id`: every bot and the account's P&L); `hosted_bot_trades` has the fills, orders
and open positions; `hosted_bot_logs` says what it did and why it stopped; `list_hosted_bots` is the
whole fleet. Report paper results as plainly as simulator results, losses included.

### Getting paid to keep orders on the book: `lip_harvester`

When the user asks about rewards, rebates or "getting paid to make markets": Kalshi's Liquidity
Incentive Program pays a daily pool per market ($1 to $1,000) to resting orders near the top of
the book, until 1 Jan 2027 unless Kalshi ends it sooner. `lip_harvester` collects it: a Yes bid
and a No bid at each side's reward reference price, on markets where the pool is large next to
what trades, sized 25/100/300 a side by reward per dollar of collateral, holding at most twice
its quote size in a market and stopping at 10% of the budget down. `markets` can be `{}` (every
market with a program: `{incentive: "lip"}`). Say it plainly: "get paid to keep orders on the
book". In `bot_report`, "Estimated LIP reward" is an estimate from Kalshi's filed formula: a
paper bot is never paid, and live the reward is Kalshi's to pay (daily, nothing under $1 a
program). Name the risks once, without dwelling on them: fills can lose money when the seller
knows more, more competing orders shrink the share, the program can end, and only US members
without a market-making agreement in that market are paid. `params: {sides: "cheap_only"}` rests
one bid on the long-shot side of markets priced ≤10¢ or ≥90¢ (much more reward per dollar in
the research); offer it only if the user asks, and say that Kalshi's filing describes the
two-sided test as preventing "one-sided order activity", so Kalshi may view it as outside the
program's purpose and can refuse rewards for it.

## The user's own algorithm: `rules`

When no built-in strategy fits the idea, write it as rules: `strategy: "rules"`, `params: {rules:
[...]}` (at most 20). Langbot checks them and runs them server-side; they are data, never code.
Each rule:

- `on` (when it is evaluated): `"book"` (every price update, the default), `{"phrase_hit":
  {counts: true, min_confidence: 0.6, phrase: "tariff", speaker: "powell"}}`, `{"score_change":
  {team: "NYK", min_points: 3}}`, `{"release": {source: "bls", contains: "payrolls", target:
  "refs"|"all"}}`, `{"calendar": {lead_min: 30}}` (from 30 minutes before the market's event to
  its end).
- `when`: a condition tree, `{"all": [...]}`, `{"any": [...]}`, `{"not": ...}` over comparisons
  `{"metric", "op" (< <= > >= == !=), "value" (a number or {"metric": ...})}`. Metrics:
  `yes_bid yes_ask no_bid no_ask mid spread bid_depth ask_depth depth hours_to_close
  minutes_to_close minutes_to_event event_live position held_yes held_no`, and over `window_s`
  (≤ 86400): `change` (mid now − then, dollars), `abs_change`, `high`, `low`, `volume`
  (trade prints: live on Kalshi's stream only; unknown on paper and in replays).
- `action`: `{side: "yes"|"no", action: "buy"|"sell", price: "1 - yes_bid" (an expression in
  the side's own price: numbers, the metrics, + − × ÷, min, max, round, abs; null = market),
  size: {contracts: n} | {usd: x} | {budget_fraction: f}, tif: "ioc"|"gtc"}`.
- `exit` (closes what the rule opened): `take_profit_c`, `stop_loss_c` (cents per contract),
  `before_close_min`, `max_hold_min`. Also `once_per_market` (default true), `cooldown_s`,
  `max_entries`.

Example ("buy No on any market priced 10¢ or less with at least 2 hours left, 20% of the budget
each, out 30 minutes before the close"):

```json
{"rules": [{"name": "fade cheap yes",
  "when": {"all": [{"metric": "yes_ask", "op": "<=", "value": 0.10},
                   {"metric": "hours_to_close", "op": ">=", "value": 2}]},
  "action": {"side": "no", "price": "1 - yes_bid", "size": {"budget_fraction": 0.2}},
  "exit": {"before_close_min": 30}}]}
```

Workflow: translate the user's idea into rules and read them back in one sentence; test them
with `sim_run(event_id, "rules", {rules: [...]}, ...)` on recorded events (same loop as above);
then `preflight_bot` (it replays the last week on recorded books and names any condition that is
false on every market); then `deploy_bot`. If `deploy_bot` answers a rules error, it names the
path (`rules[0].when.all[1].metric`): fix that part only.

## Live, on the user's own Kalshi account (US users)

Only when the user asks to trade real money. Every hosted strategy can go live, market making
and resting orders included. Everything happens here, in the chat: the user never needs the
website after the first sign-in.

1. **Ask for the budget** ("How much should it trade with? For example $20.") and, if they have
   not given it yet, **their Kalshi API key**: the key id and the private key, or the path to
   the key file Kalshi downloaded (then read the file and pass its text). If they have no key,
   tell them where to make one in Kalshi (Account, API keys) and wait.
2. Call `connect_kalshi(bot_id, key_id, private_key, budget_usd)`, with `go_live=true` when the
   user already said to go live. Say in one sentence: "I use this key once to make a trade-only
   key for the bot, and your original key gets deleted on Kalshi." Then read back
   `tell_the_user`. Defaults: the user's main Kalshi account, the normal path on every Kalshi
   account (the bot only touches its own orders, and its budget caps what it can spend), and
   `env="prod"`. `mode="subaccount"` only if the user's Kalshi account has the Advanced API
   tier and they ask for a sub-account holding just the budget. `keep_original_key=true` only
   when the user asks to keep their key.
3. If `connect_kalshi` did not go live, confirm ("Go live with $20 on Kalshi?") and, after the
   user says yes, call `promote_bot(bot_id, confirm=true)` (without `confirm` nothing goes
   live). Name any guardrail that is off. Then show `bot_report(bot_id)`.
4. The kill switch is always `stop_bot(bot_id)`; for everything at once `kill_switch()`: do it at
   once when the user says stop. A live stop cancels the bot's resting orders on Kalshi within
   seconds; open positions stay until they settle or the user closes them in Kalshi.

Never repeat the private key back in the chat or put it in a file. If `connect_kalshi` answers
`scope_denied`, the user signs in to Langbot again from this tool. The web page is an
alternative, not a requirement: `go_live(bot_id)` returns `connect_url` for a user who would
rather paste the key there; then poll `live_setup_status(bot_id, wait_s=45, since_status=…)`.

If `promote_bot` answers `live_disabled`, live trading is not open to that account yet: keep the
bot on paper and say so.

## Versions, rollbacks and fleets

Every change is a new deployment (an immutable version with its own results). `update_bot(bot_id,
budget_usd? params? markets? limits? schedule?)` makes a new paper version with only those fields
changed; `promote_bot(confirm=true)` takes a paper version live; `rollback_bot(bot_id,
deployment_id)` runs an earlier version again on the target it ran on (a live one needs the
user's yes and `confirm=true`). Several bots can run at once, each with its own
guardrails; `set_fleet_limits` adds optional caps across all live bots.

## Style

Short sentences, numbers with signs and dollars (`+$3.10`, `−$1.25`), no jargon unless the user
uses it. Losses are reported as plainly as gains. Kalshi requires users to be eligible to trade in
their jurisdiction; Langbot's live bots are for Kalshi accounts in the US.
