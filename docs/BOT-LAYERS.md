# Market trading bot layers: reference and reverse-engineering guide

Purpose: a checklist of the layers a market trading bot is made of, so that when an existing bot's repo is read, nothing gets missed, and so the original system gets designed on purpose rather than by accident.

Contents
1. The layers (what to ask, what to grep for, how it fails)
2. Reverse-engineering procedure and templates
3. Using REA (morluto/rea) to fill the table
4. Case study 1: the prediction-market video
5. Case study 2: jev-trader (this repo)
6. Transcript of the video

---

## 1. The layers

Read top to bottom. A tick of market data enters at layer 2 and an order leaves at layer 8. Layers 9 to 14 wrap around that path.

```
 0 Thesis ........ why should this make money at all
 1 Universe ...... which markets / pairs
 2 Ingestion ..... how data and time enter
 3 State ......... book, features, memory
 4 Signal ........ rules / stats / ML / LLM -> typed output
 5 Policy ........ signal -> intent (thresholds, hysteresis, abstain)
 6 Sizing ........ how much
 7 Risk .......... what is allowed (pre-trade limits, kill switch)
 8 Execution ..... order type, routing, nonce, gas, retries, cancel
 9 Accounting .... fills, position, P&L, fees, inventory
10 Custody ....... keys, wallets, permissions
11 Infra ......... hosting, RPC, config, deploy, secrets
12 Observability . logs, metrics, alerts, dashboard
13 Simulation .... backtest, paper, replay
14 Feedback ...... postmortem, calibration, tuning
```

For each layer: question to answer, grep hints for finding it in a repo, and the usual ways it breaks.

### 0 Thesis
- Question: where does the edge come from? Spread capture, latency, information, prediction, structural flow, or nothing.
- Grep: README claims, comments near the signal, "edge", "alpha", "arb", "mm", "maker".
- Fails: no stated edge, so nothing to test. Win rate quoted without payoff odds. Edge that costs more in fees and gas than it earns.

### 1 Universe
- Question: one market or many? How are markets picked and dropped (liquidity, volume, time to resolution, spread)?
- Grep: market lists, filters, `MARKET`, `symbol`, `pairs`, `whitelist`, scanner loops.
- Fails: trading illiquid markets, stale markets, markets near resolution, hardcoded addresses that change.

### 2 Ingestion
- Question: what data, from where, how fast, and what is the clock? Block-driven, timer-driven or event-driven?
- Look for: WebSocket vs polling, backstop when the socket drops, coalescing bursts, one RPC or several, read vs write endpoints, rate limits, alt data (news, social, on-chain flows).
- Grep: `WebSocket`, `subscribe`, `newHeads`, `setInterval`, `getLogs`, `rate`, `backoff`.
- Fails: stale data treated as fresh, missed blocks, out-of-order events, duplicate events, rate-limit bans, one RPC as a single point of failure.

### 3 State
- Question: what does the bot remember between ticks? Order book reconstruction, rolling windows, features, open orders, last decision.
- Grep: classes or modules holding arrays/maps updated per tick, `history`, `window`, `ring`, `cache`.
- Fails: unbounded memory growth, state not rebuilt after restart, state that disagrees with the venue (phantom orders).

### 4 Signal
- Question: what produces the view, and what is its output contract? Rules, indicators, statistical model, gradient boosting, LLM. Is the output typed (side + probability) or free text that code then parses?
- Look for: inputs given to the model, latency budget, fallback when it fails, deterministic stand-in for testing.
- Grep: `predict`, `model`, `prompt`, `schema`, `probabilities`, `confidence`.
- Fails: look-ahead leakage in features, LLM output parsing errors, model slower than the tick, no fallback, non-determinism making bugs unreproducible.

### 5 Policy
- Question: how does a signal become an intent? Thresholds, hysteresis (stop flip-flopping), ensembling, abstaining.
- Grep: `threshold`, `confidence >`, `cooldown`, `hysteresis`, `min_edge`, `skip`.
- Fails: threshold tuned on the same data it is judged on, churning at the decision boundary, abstaining so often the strategy never trades.

### 6 Sizing
- Question: fixed size, percent of bankroll, Kelly or fractional Kelly, volatility scaling, inventory-aware?
- Grep: `size`, `kelly`, `bankroll`, `notional`, `lot`, `min_order`.
- Fails: full Kelly on a noisy edge estimate, ignoring venue minimum order size, size not adjusted for current inventory.

### 7 Risk
- Question: what stops a bad trade or a bad day? Pre-trade checks (max position, max order, price band, stale-data check), session limits (drawdown, loss streak, order rate), kill switch, who can resume.
- Grep: `max_`, `limit`, `halt`, `pause`, `kill`, `drawdown`, `circuit`.
- Fails: limits that only log, limits checked after the order is sent, auto-resume after a halt, no fat-finger check, risk code that can be bypassed by a code path.

### 8 Execution
- Question: how does an intent become a fill? Order type (market, limit, post-only, IOC), price placement, cancel/replace, partial fills, retries, idempotency.
- On-chain extras: nonce management, gas limit and fee strategy, receipts and reverts, fire-and-forget vs wait, reorgs, MEV exposure, approvals and deposits.
- Grep: `sendTransaction`, `nonce`, `gas`, `postOnly`, `cancel`, `slippage`, `receipt`, `revert`, `retry`.
- Fails: nonce desync ("nonce too low" loops), double sends on retry, paying spread when you meant to earn it, gas costing more than the edge, orders left resting after a crash.

### 9 Accounting
- Question: what is the source of truth for fills, position and P&L? Own receipts, venue trade feed, or both? Are fees and gas in P&L? Realized vs unrealized? Mark price used?
- Grep: `position`, `pnl`, `entry`, `realized`, `fee`, `reconcile`.
- Fails: P&L that omits gas and fees, position drift from missed fills, double-counted fills, mixed units.

### 10 Custody
- Question: where is the key, what can it do, how much is in it? Hot wallet balance cap, separate deposit vs trading wallets, permissions.
- Grep: `PRIVATE_KEY`, `mnemonic`, `signer`, `Wallet`, `.env`, `approve`.
- Fails: key in logs or git, unlimited approvals, large balance in a hot key, no way to sweep funds out fast.

### 11 Infra
- Question: where does it run, how is it configured, deployed and restarted? Single process? Region vs RPC latency? Secrets handling?
- Grep: `Dockerfile`, `railway`, `fly`, `systemd`, `config.ts`, env parsing, health endpoints.
- Fails: config only in env with no validation, silent defaults, restart loops that resend orders, no health check.

### 12 Observability
- Question: can you tell in 10 seconds if it is healthy and what it did? Structured logs, metrics (latency, late ticks, fill rate), alerts, a dashboard, per-decision audit trail.
- Grep: `log`, `metrics`, `sse`, `/health`, `alert`, `telegram`, `webhook`.
- Fails: no per-decision record so postmortems are guesses, alerts nobody reads, dashboards that hide losses.

### 13 Simulation
- Question: backtest, paper, dry-run on live data, replay? Is the fill model honest (queue position, adverse selection, latency, fees)?
- Grep: `dry_run`, `paper`, `backtest`, `replay`, `simulate`.
- Fails: fills assumed at mid, no fees, no latency, backtest trading on data the live bot could not have seen, dry-run code path that differs from live.

### 14 Feedback
- Question: how does the system learn? Offline postmortem, online parameter tuning, prompt updates, model retraining. Is a human in the loop? How is overfitting guarded?
- Grep: `postmortem`, `retrain`, `calibrat`, `memory`, `reflect`, `lessons`.
- Fails: auto-updating prompts or params from a handful of losses, calibration measured on training data, no change log so you cannot tell which change helped.

---

## 2. Reverse-engineering procedure

Do this per repo. Aim for a filled template, not a summary.

1. Entry point: find the main loop and what triggers a tick. Note it.
2. Trace one tick: follow a single event from ingestion to order to fill to accounting. Write the files and functions touched in order.
3. Fill the layer table below, one row per layer. "Absent" is a valid and useful answer.
4. List every config knob and its default. Defaults reveal the author's real assumptions.
5. Grep failure handling: `catch`, `retry`, `timeout`, `halt`, `TODO`, `FIXME`, `hack`. This shows the layers they had to bolt on after something broke.
6. Check claims vs code: README/tweet/video claims against what the code actually does.
7. Verdict: what to copy, what to avoid, what is missing that you would want.

### Layer table (copy per repo)

| # | Layer | Present? | Where (file:line) | How it works | Weakness | Steal / avoid |
|---|-------|----------|-------------------|--------------|----------|---------------|
| 0 | Thesis | | | | | |
| 1 | Universe | | | | | |
| 2 | Ingestion | | | | | |
| 3 | State | | | | | |
| 4 | Signal | | | | | |
| 5 | Policy | | | | | |
| 6 | Sizing | | | | | |
| 7 | Risk | | | | | |
| 8 | Execution | | | | | |
| 9 | Accounting | | | | | |
| 10 | Custody | | | | | |
| 11 | Infra | | | | | |
| 12 | Observability | | | | | |
| 13 | Simulation | | | | | |
| 14 | Feedback | | | | | |

### Tick trace (copy per repo)

```
trigger:
  -> ingestion:
  -> state update:
  -> signal:
  -> policy:
  -> sizing:
  -> risk check:
  -> execution:
  -> receipt / fill:
  -> accounting:
  -> log / stream:
```

### Verdict (copy per repo)

```
Edge claimed:            Edge actually in code:
Strongest layer:         Weakest layer:
Layers absent:
Config defaults that surprised me:
Copy:
Avoid:
Questions for the author:
```

### Cross-bot comparison

After 3 or more repos, make one grid: rows are layers, columns are bots, cells are one-line answers. The empty and divergent cells are the design decisions for the original system.

---

## 3. Using REA (morluto/rea) to fill the table

What REA is: "Reverse Engineer Anything", an MCP server and CLI (`npx rea-agents setup`) that lets an agent inspect software it has no source for and return findings with evidence. It is not a collection of bots. It is the tool you point at a bot. Local analysis only. Needs Node 22+.

Read first: `DISCLAIMER.md` in that repo. It excludes circumventing protections, bypassing access controls on systems you are not authorized to test, and copying proprietary code. Use it to learn which layers a bot has and how they are wired, not to clone a product. Possession of a file or a public endpoint is not permission. Check the target's license and terms before running anything.

### When you do not need it

Open-source bot: just read the repo with the procedure in section 2. REA adds nothing over reading source.

### Which REA target answers which layer

| Bot form | REA capability | Layers it exposes |
|----------|----------------|-------------------|
| Electron or desktop app bot | JavaScript / Electron analysis: modules, imports, routes, IPC, source maps | 2 ingestion (what endpoints and feeds), 4 signal and 5 policy (if logic runs client side), 6 sizing, 7 risk limits, 8 execution calls, 11 config |
| Web dashboard or web bot UI | Website analysis (scripts, network observations); HAR/SAZ network captures | 2 ingestion, 8 execution (which APIs, payload shapes), 12 observability (what the UI reports) |
| Python bot shipped as an executable | PyInstaller analysis: Python version, entry scripts, bytecode | Everything, since the strategy lives in the bytecode. Highest yield and most legally sensitive |
| Go or Rust binary | Embedded module and crate lists, build settings | 2, 8, 10, 11 from dependencies (which exchange SDKs, signing libs, RPC clients) |
| Native binary | Hopper, Ghidra or IDA pseudocode, strings, calls | Same as above, much slower |
| Local state DB | SQLite inspection: tables, columns, rows | 3 state, 9 accounting, 12 logs. Schema shows what the author thought worth persisting |
| On-chain bot contract | EVM bytecode: selectors, dispatch, mutability | 7 risk (access control, caps), 8 execution (what functions exist) |
| Android app | APK, ADB | Same as Electron |
| Traffic between bot and venue | PCAP summaries, packet analysis | 2 ingestion cadence, 8 execution rate and retry behavior |

### What REA cannot show

- Server-side logic. A hosted signal, policy or risk engine behind an API is invisible. All you see is the request and response shape.
- Edge and results. Code tells you what a bot does, never whether it makes money.
- For on-chain bots, the best ground truth is not REA. It is the wallet's transaction history: order cadence, size distribution, cancel/replace ratio, gas paid per order, inventory over time. Read that with an explorer or logs, then use REA only to explain why.

### Workflow per target

1. Pick the form from the table. Note the authorization basis in one line (own software, open source, permissive license, public interop, research scope).
2. Run the matching REA analysis. Save the evidence output.
3. Fill the layer table from evidence. Mark each cell `seen`, `inferred` or `absent`. Do not let `inferred` pass as `seen`.
4. Cross-check against observable behavior (wallet history, UI output, README claims). Disagreements are the interesting findings.
5. Write the verdict block. Keep patterns and ideas; do not carry over code, assets or proprietary logic.

### Layer table with provenance (use instead of the plain table when working from REA)

| # | Layer | Evidence | Confidence (seen / inferred / absent) | Source (REA tool, file, wallet tx) |
|---|-------|----------|----------------------------------------|-------------------------------------|

### Setup note

`npx rea-agents setup` edits your agent configs (with backups, after approval). I have only cloned and read the repo here; I have not installed or run it. The terminal route needs no setup: `npx -y rea-agents@latest analyze-javascript-application /abs/path/to/app --json`. Most real work needs Hopper, Ghidra or IDA on your machine, which this cloud session does not have. Run REA locally and bring the evidence output here for me to turn into layer tables.

---

## 4. Case study 1: the prediction-market video

Source: https://youtu.be/shZwbIRtObE (YouTube Short, 1:09, Ray Fu, 2026-05-02). Read from the auto transcript and page metadata; visuals not seen.

Source quality: lead generation (comment for the guide, then a consulting pitch). "$1,500 a day" and "68.4% win rate" come with no backtest, code or record. A win rate without payoff odds is not an edge. The "33-page Anthropic cheat sheet" is unverified. Treat as an outline only.

Mapped to the layers:

| Video stage | Layers | Notes |
|-------------|--------|-------|
| 1 Scan agent (300+ markets, liquidity, volume, time to resolution, odd moves, wide spreads) | 1 Universe | Filter on tradability before looking for signal |
| 2 Research agents (Twitter, Reddit, RSS, sentiment vs market odds) | 2 Ingestion, 4 Signal | Alt-data input. Slow, so suits slow markets only |
| 3 Prediction agent (XGBoost + LLM, true probability vs price, fire above threshold) | 4 Signal, 5 Policy | Edge = model probability minus market price. Threshold is the policy |
| 4 Risk agent (size from edge and bankroll, block if too big, place on chain, watch to settlement) | 6 Sizing, 7 Risk, 8 Execution, 9 Accounting | Kelly-style sizing, pre-trade block, settlement tracking |
| 5 Postmortem (five agents after each loss, save, update the system) | 14 Feedback | Loss-triggered only, which biases learning. Auto-update is the danger |

Not mentioned at all: thesis proof (0), custody (10), infra (11), observability (12), simulation (13). That absence is itself the lesson: demo-grade pipelines describe 4 to 5 layers, real bots need all 15.

Fit to jev-trader: stages 1, 2 and 3 conflict with the CLAUDE.md non-negotiables (Jev decides from the price feed, an order every block, never decide every N blocks). Stage 4 maps to a drawdown halt that never overrides Jev's side. Stage 5 maps to an offline report on logged sessions with a human editing the prompt.

---

## 5. Case study 2: jev-trader (this repo)

Mapped from file headers and the README; line numbers not yet verified. Treat "where" as pointers to read next.

| # | Layer | Where | State |
|---|-------|-------|-------|
| 0 | Thesis | README, SPEC.md | Demo, not a profit claim. Post-only quoting earns the spread; footer says the model is not trying to be profitable |
| 1 | Universe | `config.ts` `MARKET` | One market: MON-USDC on Kuru |
| 2 | Ingestion | `chain.ts`, `book.ts`, `trades.ts` | WebSocket newHeads with HTTP polling backstop, coalesced to newest block. One raw `eth_call` for the book, `getLogs` for trades |
| 3 | State | `trader.ts` | Rolling mid history sampled every 5 blocks over the horizon, resting orders, position |
| 4 | Signal | `model.ts` | Jev via AI SDK (typed buy/sell with probabilities) or a deterministic mock stand-in |
| 5 | Policy | `trader.ts` | Side hysteresis (flip only past `FLIP_THRESHOLD`), `hold` only on a late block |
| 6 | Sizing | `config.ts` | Fixed `TRADE_SIZE_MON` (venue minimum 200 MON) |
| 7 | Risk | `config.ts`, `trader.ts` | `MAX_POSITION_MON` cap pushes the quote to the other side. No drawdown halt yet (gap) |
| 8 | Execution | `market.ts` | Post-only limit, one `batchUpdate` that also cancels the last order, hardcoded gas limit, nonce resync to the higher of confirmed and pending, fire-and-forget with receipts reported later |
| 9 | Accounting | `market.ts`, `trades.ts`, `trader.ts` | Maker fills from the Trade log feed, position, realized and unrealized P&L, gas in MON and USD, Jev spend |
| 10 | Custody | `.env` | Single hot key, `DRY_RUN` default. Margin account funding at startup |
| 11 | Infra | `Dockerfile`, `config.ts`, `server.ts` | Single Bun process, deployed on Railway, SSE stream plus Next.js dashboard in `web/` |
| 12 | Observability | `server.ts`, `web/` | Per-block event stream, `/history`, dashboard. No persisted log yet (gap) |
| 13 | Simulation | `DRY_RUN` | Real book and real decisions, simulated fills when a real print crosses. Replay is in SPEC.md, not confirmed built |
| 14 | Feedback | none | Gap. Needs a session log first |

Gaps worth closing first: a persisted per-block log (12 and 14 both depend on it), a drawdown halt (7), then an offline postmortem report (14).

---

## 6. Transcript of the video (verbatim)

> I wish someone told me this sooner, but you can actually use a cloud skill to build out prediction market bots that can make you $1,500 a day. And Tropic recently released a 33-page cheat sheet for building out cloud skills. But hidden within it is a prediction market trading bot that has a win rate of 68.4%. Here's how it works in five steps so you can build it out yourself.
>
> Step one, a scan agent filters over 300 active markets and filters by liquidity, volume, and time to resolution. It flags anything with weird price moves or wide spreads.
>
> Two, research agents run in parallel scraping Twitter, Reddit, and RSS feeds. They run sentiment analysis and compare the narrative against market odds.
>
> Step three, a prediction agent uses XG Boost along with an LLM to calibrate true probability versus market price. It only fires when confidence is above a threshold.
>
> Step four, before it places any bet, a risk agent determines if the bet is too big for your account. It calculates exactly how much to put in based on your edge and your bankroll. If the trade is too risky, it blocks it, but if it passes, it places the trade on chain and watches until settlement.
>
> Step five, after every loss, five agents run postmortem. They figure out what went wrong, save it, and update the system so it doesn't make the same mistake twice.
>
> If you want a link to the guide detailing this, just come and trade and I'll send it to you. If you want me for a customized solution with AI to help automate your business or make you more money, just come and scale and we can set a meeting to talk. Follow for more advice and news on how to make money with tech and AI.

(Speech-to-text slips: "cloud skill" = Claude skill, "Tropic" = Anthropic, "XG Boost" = XGBoost.)
