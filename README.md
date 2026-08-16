# Trading Ladder Tracker

A single self-contained HTML file. No build step, no dependencies, no network calls.

## Use it

**<https://acgroup21.github.io/Trading/>** — no download, no install. Log each trade
by what you put in and either what you finished with or the multiplier you got, and
the two fill each other.

You can still open `trading-tracker.html` from disk if you prefer; it behaves the same.

## Where your data lives

In your browser, on the machine you're using. Nothing is sent anywhere.

Storage belongs to the **site**, not the file. On the live link above every page
shares one store, so it just works and keeps working. Local copies are the opposite:
each file opened from disk is its own island, and a second copy under a different
filename starts empty — so if you work from disk, keep one file and overwrite it in
place rather than saving numbered versions.

Moving between the two, or between machines, is what **Download my data** and
**Load a data file** are for. Do that before switching, not after.

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
