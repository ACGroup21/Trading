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

## Start with the plan

It opens empty on purpose. Five numbers decide the outcome more than any single trade
does — starting capital, the share of the pot you stake, the payoff you aim for, the
share you bank, and the goal — so the app asks for them rather than arriving with
opinions pre-filled. Logging is locked until they are set.

Not sure yet? **Show me a worked example** loads a demo: the recommended configuration
and four trades that are deliberately not a clean sweep, so it shows the difference
between a win and a hit. One click clears it back to empty.

## What it does

| | |
|---|---|
| Your plan | The five numbers in one place, stated once, then held to. Per-trade settings still win for their own trade, so changing the plan never rewrites history |
| Position sizing | What share of the pot goes in — a loss costs the stake, not the ladder |
| Banking | What share of each close is locked away and never traded again; both stored per trade |
| Sticking to the plan | Flags stakes sized off plan and winners closed short, and prices the early exits in **rungs lost**. Thresholds are yours to set |
| Costs and tax | Spread each way, slippage as the stake grows, commission, and tax as a liability against the safe haven. Applied to forecasts only — a logged trade already includes what it cost |
| Projection | Models fractional deployment; any line can be locked in as a fixed milestone |
| Odds | Monte Carlo probability of reaching your goal at your realised hit rate, next to the same ladder staked all-in |
| Record | Hit rate against target (not against breakeven), streaks, drawdown, per-market breakdown |
| Cash flow | Deposits and withdrawals recorded separately so growth isn't inflated by money you paid in |
| Saved plans | Keep several side by side — a live ladder and a what-if. Held apart from the working tracker, so a reset can't take them |
| Markets | 1,114 instruments — indices, US/UK/Europe stocks, top-100 crypto with CoinMarketCap links, forex, commodities, ETFs |

## What it won't do

It won't tell you the plan is good. Every probability it reports is conditional on a hit
rate that only your own logged trades can establish, and it says so rather than quietly
assuming one. Turning costs on makes the projections look worse; that is the same plan
described more accurately, not a plan that got worse.

Not financial advice, and not tax advice. It does the arithmetic of the plan and the
rates you give it; whether that plan is achievable, and how it is taxed, are separate
questions for you and your accountant.
