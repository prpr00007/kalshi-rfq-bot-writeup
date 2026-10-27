# Kalshi RFQ Market-Making Bot

An automated market maker for multi-leg combo contracts on [Kalshi](https://kalshi.com), a CFTC-regulated prediction-market exchange. It reads every quote request (RFQ) on the exchange, prices the non-sports combos with a correlation model, sends two-sided quotes, and confirms fills inside Kalshi's 3-second acceptance window, all behind layered risk controls.

**The source is private because the bot trades real money.** This page covers the system design, how it was tested, and what went wrong along the way. It leaves out the pricing parameters and risk limits.

`Python` `asyncio` `NumPy` `SQLite` `Docker` `Prometheus` `Grafana` `Alertmanager` `GitHub Actions`

---

## The problem

On Kalshi, anyone building a parlay ("combo") in the app sends an RFQ to every API user, and market makers race to quote it. Combos have no order book, so the RFQ is the only way they trade.

Most of that flow is sports parlays. About 3–4% is non-sports, mostly 15-minute crypto baskets like "BTC up AND ETH up AND SOL up in the next 15 minutes", plus some commodities. That is still millions of requests a day. Other market makers already fill them, so the question wasn't whether anyone quotes but whether there is room to quote better.

The obvious way to price a combo is to multiply its legs' probabilities. That's wrong for crypto, because the coins move together. In 30 days of Kalshi history, "all 9 coins up in the same 15 minutes" happened **14.8%** of the time, against **0.2%** if they were independent. A maker who multiplies legs gets picked off on exactly the combos takers like best.

## Architecture

```
                 Kalshi WebSocket feeds (RFQs, trades, order-book updates)
                                      │  peaks ~5,000 messages/s
                                      ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ listener   classify each RFQ from its legs (one series download,     │
   │            no per-RFQ lookups) · per-minute counts · hash-sampled    │
   │            detail to SQLite · live leg-price cache                   │
   └───────────────┬──────────────────────────────────────┬───────────────┘
                   │ non-sports combos                     │ settlements
                   ▼                                       ▼
   ┌──────────────────────────────┐         ┌───────────────────────────────┐
   │ pricing   Gaussian copula     │         │ settle / history              │
   │ fitted on 43,237 settled      │◀────────│ 15-minute results, refit      │
   │ 15-min windows, 100,000-path  │         │ the correlation model         │
   │ Monte Carlo, ~2 ms per RFQ    │         └───────────────────────────────┘
   └───────────────┬──────────────┘
                   ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ quoter    price → risk check on the whole trade (quotes are all or   │
   │           nothing) → two-sided quote                                 │
   │           on acceptance: re-price with fresh legs, re-check limits,  │
   │           confirm or decline (median 49 ms acceptance → confirm)     │
   │           verify every fill against what was expected                │
   └───────────────┬──────────────────────────────────────────────────────┘
                   ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │ risk + account   per-trade, open-exposure, per-settlement-window,     │
   │                  daily-loss and drawdown limits · feed-health pause · │
   │                  kill switch · reconcile with Kalshi every 30 s       │
   └───────────────┬──────────────────────────────────────────────────────┘
                   ▼
     Prometheus metrics → Grafana dashboard · Alertmanager + Telegram alerts
```

- **Async and event-driven.** One asyncio process holds the WebSocket connections, keeps a live cache of leg prices, and prices each RFQ the moment it arrives. Database writes are buffered so disk I/O never sits in the quoting path.
- **Classification without lookups.** The first version looked up every RFQ over REST and would have fallen hours behind. The current one classifies each RFQ from the legs in the message, using one download of Kalshi's series list. It handles about 240,000 messages a second in benchmarks, far above the feed's peak.
- **Correlation model.** A Gaussian copula fitted to the correlations of 15-minute returns, with 100,000 simulated paths. Combos with legs it doesn't cover aren't quoted at all, rather than falling back to a worse model.
- **Last look.** When a requester accepts, the bot has 3 seconds. It re-prices with the latest leg prices and confirms only if the trade is still worth taking and every limit still holds. Across 436 live confirmations, the median time from acceptance to confirmation was **49 ms**.
- **The exchange is the source of truth.** Every 30 seconds the bot adopts Kalshi's balance and open positions. Any fill that doesn't match what it expected trips the kill switch.
- **Least-privilege keys.** The live bot uses an API key with only read and trade permissions, so it cannot move money out of the account.

## How it was validated

Each stage had to work before any money went in:

| Stage | What it does | Money at risk |
|---|---|---|
| 1. Listener | Two weeks of recorded RFQs, fills and settlements; measures what existing makers earn | none |
| 2. Historical replay | Re-runs settled fills as if we had quoted, after fees | none |
| 3. Shadow quoting | The pricing policy is **locked**, then judged only on data recorded after the lock | none |
| 4. Paper trading | Prices every live RFQ in real time through the same risk manager the live bot uses | none |
| 5. Live pilot | Small account, tight caps, kill switch, a separate trade-only API key | small |

**Testing and CI/CD.**
- Unit tests cover request signing, classification, the model, fees, settlement, the risk manager, account rebuilds, metrics and alerts.
- A **replay regression test** feeds 30 seconds of real recorded feed (about 10,000 messages) through the trader and checks that every decision still matches the recorded expected output: priced or skipped and why, prices, fills won, and account decisions. A strategy change has to come with a deliberate update to that file.
- GitHub Actions runs lint, the unit tests and the replay test on every push. It then builds the Docker image and validates the Prometheus, Alertmanager and Compose configs. A manual job deploys to a VPS over SSH.

## Lessons learned

**The "empty market" wasn't empty.** Earlier research said nobody answers non-sports RFQs, but it had only tested single markets. The live feed showed combos are requested millions of times a day and already filled. That changed the project from finding an empty market to pricing better than the makers already there.

**The feed is a firehose.** The first listener did a lookup per RFQ and would have fallen hours behind and filled the disk. The fix was classifying from the message alone, counting everything per minute, and storing full detail only for a hash-based sample of markets.

**The first good backtest was biased twice.** It used leg prices from the moment of the fill, a second after makers had to quote. Its settings were also tuned on the same data it was judged on. The fix was to record leg prices when each RFQ arrives, lock the policy, and judge it only on data recorded afterwards.

**Silent fallbacks are expensive.** Combos with legs the model didn't cover quietly fell back to multiplying legs. They lost a large share of the first paper session. Now anything the model can't price is skipped.

**Two tests disagreed because prices move faster than snapshots.** Paper trading and the replay agreed on most fills but split on a few hundred. The cause turned out to be timing, not bugs. Legs sometimes jump 25–50 cents in a second or two, and each process sat a slightly different fraction of a second behind the feed, so their snapshots landed on different sides of the jump. That is also the biggest real-money risk, getting picked off by a stale price, so it led directly to new rules about when not to quote.

## Monitoring

- **Prometheus** tracks RFQ and fill rates, skip reasons, pricing time, feed delay, equity, daily P&L, drawdown and pauses.
- **Grafana** shows them on a provisioned dashboard.
- **Alertmanager** fires on what the bot can't report about itself: a process down, a lagging or disconnected feed, no RFQs, or a long pause.
- **Telegram** sends pause notices and accepts `/status`, `/stop` and `/resume` from one chat.

---

Built by [Prabu Ramanababu](https://github.com/prpr00007) · [LinkedIn](https://www.linkedin.com/in/praburamanababu) · happy to walk through the code in an interview.
