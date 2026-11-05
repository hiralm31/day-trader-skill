# ChatGPT + Robinhood analysis adapter

This directory documents how to use the day-trader methods in ChatGPT with the
connected Robinhood MCP tools. It is an analysis adapter, not a Claude skill
installer or an autonomous trading bot.

## Current operating mode: analysis only

The configuration in `config.robinhood.analysis.json` is a proposed baseline:

- Capital: $10,000
- Maximum planned loss per trade: $50 (0.5% of capital)
- Maximum planned daily risk: $150 (1.5% of capital)
- Maximum new entries per day: 3
- Mode: analysis only
- Live order submission: disabled
- Options, margin, short selling and crypto: disabled

The inherited defaults (10% cash buffer, 75% position cap, $4,000 max order,
$5 minimum order, 1R minimum reward/risk, 09:45–15:30 ET entry window, 15:55 ET
flat-by target) are conservative workflow defaults from the original sample
configuration, not separately confirmed user preferences. Revisit them before
using the configuration outside analysis mode.

## ChatGPT tool mapping

| Original capability | ChatGPT / Robinhood analysis path |
| --- | --- |
| Market quotes | Robinhood `get_equity_quotes`; identify symbols first when needed |
| Fundamentals and session OHLCV | Robinhood `get_equity_fundamentals` |
| Historical bars | Robinhood `get_equity_historicals`, requesting completed bars and an explicit interval |
| Market scanner | Robinhood scanner filter specs + `preview_scan`; use `get_scans` / `run_scan` for saved screens |
| Position sizing | Recalculate from the latest confirmed entry, stop, capital, risk cap, position cap, order cap and cash buffer |
| Risk gate | Independently validate stop direction, max loss, reward/risk, daily risk already committed, trade count, liquidity/data freshness and market hours |
| Backtesting | Use historical bars returned by Robinhood; state sample size, interval, missing bars, assumptions, seed and limitations. Do not imply Monte Carlo results are validated until the data and implementation have been checked. |

The original scanner's daily-close and opening-range rules are useful strategy
definitions, but Robinhood's scanner has its own supported filters and data
semantics. Validate filters before using them. Never claim to have scanned
thousands of symbols unless the tool's returned coverage proves that.

## Required analysis workflow

1. Confirm the market date/time and whether the market is open.
2. Run a validated market screen; report the scan scope, matching count, data
   time, and any partial coverage.
3. For a candidate only, fetch recent quotes, fundamentals and completed
   historical bars. Treat quotes as snapshots, not guaranteed execution prices.
4. Describe the setup as a rule match, not a buy signal. Check news/catalysts
   and earnings dates where reliable data is available.
5. Define entry, stop, target, expiry/flat-by time and the assumptions before
   sizing.
6. Size whole shares using the smaller of:
   - floor($50 / risk per share),
   - remaining position-cap dollars / entry,
   - remaining max-order dollars / entry,
   - available cash after the cash buffer / entry.
7. Reject the setup if one share exceeds the $50 risk cap, the stop is missing
   or on the wrong side, reward/risk is below 1.0, data is stale/incomplete, or
   the daily limits would be exceeded.
8. Daily risk accounting is conservative: count the full planned stop risk of
   each entry for the day, regardless of whether a prior trade later wins.
   Stop proposing new entries at $150 committed risk or after 3 entries.
9. Present analysis only. Never call a Robinhood order placement, review,
   approval, cancellation, watchlist-write, alert-write, or other state-changing
   tool as part of this project unless the user separately requests that
   specific action and it is permitted.

## Position sizing formula

For a long trade:

- Risk/share = entry - stop
- Shares by risk = floor($50 / risk/share)
- Shares by position cap = floor((capital × position cap %) / entry)
- Shares by order cap = floor(max order dollars / entry)
- Shares by cash = floor((cash - cash buffer) / entry)
- Final shares = the minimum of those limits

Estimated loss at stop = final shares × (entry - stop). A stop is not a
guaranteed fill; gaps and slippage can make actual losses larger.

Example only: entry $100, stop $98 -> $2 risk/share -> 25 shares maximum by
the $50 risk limit, before applying the other caps.

## Backtesting guardrails

- Use only completed historical bars; do not include the current unfinished bar.
- Record the exact data range, interval, session bounds and adjustment mode.
- Do not silently fill missing bars or treat interpolated bars as observations.
- For bars where stop and target are both touched, use the conservative stop
  first assumption unless finer-grained data resolves the sequence.
- Separate historical replay from Monte Carlo simulation.
- Do not present simulated frequencies as probabilities of future success.
- Report transaction costs and slippage as unknown if not modeled.
- Do not produce a performance claim from a small or incomplete sample.

## Explicit non-goals

- No live order execution.
- No autonomous trade placement.
- No transfer of credentials, tokens or account secrets into repository files.
- No claim that this ChatGPT guide installs native persistent skills.
- No automatic use of the original Alpaca-specific `trader-broker` adapter.

The original Claude/Alpaca broker code remains untouched. This guide only
defines the analysis workflow until a separately reviewed broker adapter is
requested.
