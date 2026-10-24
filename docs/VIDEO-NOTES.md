# Video notes: "How to build prediction trading market bots with Claude"

Source: https://youtu.be/shZwbIRtObE (YouTube Short, 1:09, Ray Fu, 2026-05-02, ~312k views)
Note: I read the auto transcript and page metadata. I did not see the visuals. A 70 second Short is almost certainly a talking head with no code on screen.

## Transcript (verbatim)

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

## Source quality

Treat this as an outline, not evidence.

- It is lead generation. The CTA is "comment a keyword and I'll DM the guide", then a consulting pitch.
- "$1,500 a day" and "68.4% win rate" come with no backtest, no live record, no code, no sample size.
- A win rate alone is not an edge. On prediction markets, buying a 90 cent outcome wins 90% of the time and still loses money if the odds are wrong. Without payoff odds the number is meaningless.
- The "33-page Anthropic cheat sheet" is unverified. I did not find or open it.
- The architecture itself (scan, research, predict, size, postmortem) is a standard pipeline and is fine as a shape.

## Notes: the five steps

| # | Stage | What it does | Tech named |
|---|-------|--------------|------------|
| 1 | Scan agent | Filter 300+ markets by liquidity, volume, time to resolution. Flag odd price moves and wide spreads | none |
| 2 | Research agents (parallel) | Scrape Twitter, Reddit, RSS. Sentiment vs market odds | LLM sentiment |
| 3 | Prediction agent | Estimate true probability vs market price. Fire only above a confidence threshold | XGBoost + LLM |
| 4 | Risk agent | Size from edge and bankroll. Block if too large. Place on chain, watch to settlement | Kelly-style sizing (implied) |
| 5 | Postmortem (5 agents, after each loss) | Diagnose, save, update the system | memory / prompt updates |

## Does it fit jev-trader?

Short answer: only two of five stages fit. The video is a many-market prediction-market bot. jev-trader is one market (MON-USDC on Kuru), one model call per block, and CLAUDE.md fixes four claims that must stay true: Jev makes the buy/sell call from the price feed, real wallet, an order every 300 ms block, the live dashboard is the demo.

| Video stage | Verdict | Why |
|-------------|---------|-----|
| 1 Scan | Drop | One market. Nothing to scan |
| 2 Research agents | Drop | Jev decides "from the price feed". Social scraping breaks the claim and cannot finish inside 300 ms |
| 3 XGBoost + threshold | Drop | Code would make the call, and "only fire above a threshold" skips blocks. Both break the non-negotiables. Do not decide every N blocks either |
| 4 Risk agent | Keep, narrowed | Sizing is already fixed (`TRADE_SIZE_MON`, `MAX_POSITION_MON`). Add a drawdown halt that reuses the existing "paused" state. It never overrides Jev's side |
| 5 Postmortem | Keep, offline | Runs after the session on logged data. A human edits the prompt. Nothing self-modifies the live loop |

Also keep one idea from stage 3 in a legitimate form: measure how well calibrated Jev's probabilities are (Brier score of buy probability vs realized move over `HORIZON_BLOCKS`). That is evaluation, not decision making.

## Plan

Phase 0 (no code): confirm the live loop is untouched. Jev decides, every block, from the price feed.

Phase 1: session log
- Append one JSONL line per block from `src/trader.ts`: block, mid, bid, ask, decision, probabilities, latencyMs, late, fill, position, gasMon.
- Gitignored `logs/` directory. Needed by everything below.

Phase 2: drawdown guard (`src/trader.ts`, `src/config.ts`)
- New env: `MAX_DRAWDOWN_PCT` (of `BANKROLL_USD`), `MAX_LOSS_STREAK_BLOCKS` optional.
- When tripped: state becomes paused. Decisions keep running in dry-run, shown greyed (already in SPEC.md "Out of funds / paused"). No fills.
- Fails loud: log the reason and push it in the stream so the banner says why.

Phase 3: postmortem script (`scripts/postmortem.ts`, run with `bun scripts/postmortem.ts logs/<session>.jsonl`)
- Reads the log, writes a markdown report. No LLM needed for v1.
- Metrics: fill rate per side, mean markout at 1, 10, 100 blocks per fill, spread captured vs gas paid, late block rate, p95 latency, calibration table and Brier score of buy probability.
- Optional v2: feed the report plus the 20 worst blocks to Claude and ask for prompt changes. Output is a suggestion file. A human applies it to `src/model.ts`.

Phase 4: act on findings (manual, one change per session): prompt wording, `QUOTE_INSIDE_TICKS`, `TRADE_SIZE_MON`, flip hysteresis. Re-run and compare reports.

## Designs

### A. Mapping of the video pipeline onto jev-trader

```
VIDEO                               JEV-TRADER
scan 300 markets        ----X       single market, nothing to scan
research agents         ----X       price feed only
XGBoost + threshold     ----X       Jev decides, every block
risk agent              ----->      drawdown guard (pause, never override)
postmortem agents       ----->      offline report from the session log
```

### B. Per-block loop (unchanged core, new parts marked)

```
new block (300 ms)
  |
  +--> read Kuru book + recent trades
  |
  +--> Jev (typed decision, ~100 ms)  -> {buy|sell, probs}
  |       late? -> hold, count late block
  |
  +--> [NEW] guard: drawdown tripped? --yes--> paused: dry-run only
  |                                    no
  +--> post-only limit order on Jev's side
  |
  +--> stream event to dashboard
  +--> [NEW] append JSONL line to logs/<session>.jsonl
```

### C. Guard states

```
LIVE --drawdown >= MAX_DRAWDOWN_PCT--> PAUSED (dry-run, greyed, reason shown)
PAUSED --restart with new bankroll or raised limit--> LIVE
```

No automatic resume. Resuming a losing bot on its own is how bankrolls go to zero.

### D. Log line

```json
{"block":105416201,"ts":1761300000000,"mid":0.0231,"bid":0.0230,"ask":0.0232,
 "action":"buy","p":{"buy":0.58,"sell":0.42},"latencyMs":94,"late":false,
 "fill":{"side":"buy","size":200,"price":0.0230,"tx":"0x..","gasMon":0.03},
 "pos":{"side":"long","size":400,"entry":0.0229}}
```

### E. Postmortem report layout

```
# Session <id>   blocks N   fills N   late N (x%)
## P&L          pnl MON / %, gas MON, spread captured
## Markout      table: horizon (1, 10, 100 blocks) x side -> mean, hit rate
## Calibration  buy-prob bucket -> realized up rate, Brier score
## Latency      p50 / p95 / p99, late rate
## Worst 20     block, decision, probs, what the price did next
## Suggestions  (human reviews, then edits src/model.ts)
```

## Open items

- I could not get the guide the video promises. If you have it, share it and I will check the pipeline against it.
- Decide whether the drawdown guard is worth it for the demo. It adds a visible "paused" state that could undercut "an order every block". My recommendation: ship it, because an unattended bot with a live key and no stop is the bigger risk.
