# Trading Ladder Tracker

A single self-contained HTML file. No build step, no dependencies, no network calls.

## Use it

Open `trading-tracker.html` in a browser. Log each trade by what you put in and
either what you finished with or the multiplier you got — the two fill each other.

## Where your data lives

Saved data is keyed to the file's **exact path**. Open a second copy under a
different filename and it starts empty. Keep one canonical file and overwrite it
in place; use **Download my data** for backups and to move between machines.

## What it does

| | |
|---|---|
| Position sizing | Choose what share of the pot goes in (default 50%) — a loss costs the stake, not the ladder |
| Banking | Choose what share of each close is locked away (default 33%); both stored per trade |
| Markets | 1,114 instruments — indices, US/UK/Europe stocks, top-100 crypto with CoinMarketCap links, forex, commodities, ETFs |
| Projection | Models fractional deployment; any line can be locked in as a fixed milestone |
| Odds | Monte Carlo probability of reaching your goal at your realised hit rate, next to the same ladder staked all-in |
| Record | Hit rate against target (not against breakeven), streaks, drawdown, per-market breakdown |
| Cash flow | Deposits and withdrawals recorded separately so growth isn't inflated by money you paid in |

Not financial advice. It does the arithmetic of the plan you give it; whether that
plan is achievable is a separate question.
