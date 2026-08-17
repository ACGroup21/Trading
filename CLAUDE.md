# CLAUDE.md

Context for working on this repo. Read before changing `trading-tracker.html`.

## What this is

A trading journal for a **ladder strategy**: stake part of a pot, aim for a payoff
multiple, lock part of each win away in a "safe haven" that is never traded again,
roll the rest into the next trade. The tracker records real trades against that
structure and tells the owner how the plan is actually going.

One file. `trading-tracker.html` — HTML, CSS and JS inline, no build step, no
dependencies, no network calls at runtime.

## Hard constraints

- **Single self-contained file.** No bundler, no CDN, no imports. If you need a
  library, write the twenty lines instead.
- **No network at runtime.** Market lists are baked in. CoinMarketCap appears only
  as outbound `<a href>` links the user clicks.
- **ES5-flavoured JS.** `var`, `function`, no arrow functions or template literals
  in the app code. It is consistent throughout; match it.
- **Never write a C0 control character into the file.** A stray `\x01` once ended up
  inside a string literal and produced an option value nothing could match.

## The engine

State lives in one object `S`, persisted to `localStorage` under `tradingTracker.v2`
through `store()` / `fetchStore()`, both wrapped in try/catch so a sandboxed context
degrades to in-memory rather than throwing.

`S.entries` is an ordered list of two shapes:

```js
{k:"trade", date, mkt, start, finish|null, split, target, deploy}
{k:"cash",  date, dir:"in"|"out", pot:"capital"|"safe", amount}
```

`compute()` replays the whole list from `S.capital` and returns running pot, safe
haven, open stake, invested, equity and peak. **Everything derives from this replay** —
there are no stored running totals to fall out of sync.

`splitClose(start, finish, sp)` is the one place the banking rule lives:

```
bank = finish * sp
if protect: bank = finish > start ? min(bank, finish - start) : 0
roll = finish - bank
```

A trade closes as `pot = pot - stake + finish - bank`. That form is deliberate: it
handles partial deployment, and collapses to the old full-pot behaviour when
`stake === pot`.

## Decisions that are not obvious from the code

**`split`, `target` and `deploy` are stored per trade, not read from settings.**
They were global once. Changing the default silently recalculated every historical
trade and the owner's whole record shifted. Settings now supply the *default for the
next trade*; each entry keeps what it was logged with. `adopt()` stamps older files
on load so opening a new version never moves past numbers. **Do not "simplify" this
back to a global.**

**A "hit" is reaching the trade's target multiple. A "win" is making money.**
They are different and the tracker reports both. A 2.00x when the rung needed 5.00x
is profit and a miss, and the ladder only cares about the miss. `hitStats()` ignores
trades with no target set rather than assuming one.

**The safe haven is locked and cannot re-fund the pot.** This is the strongest part
of the strategy — it means the plan cannot go to zero overall. Withdrawals from it
are recorded as `cash` entries so they are visible, never as an automatic top-up.

**Deployment fraction is the dominant variable.** Simulated: same trader, same hit
rate, staking 100% of the pot gives roughly a 0.2% chance of completing the ladder;
staking 50% gives ~99%. The identity is `f = 2/(m-1)` — deploying half into a 5x
produces the same rung progression as all-in into a 3x, but a loss costs half the pot
instead of the plan. 50% deploy / 5.00x goal / 33% skim is what that reasoning
recommends — but see below: it is the demo's configuration, not a default the app
applies for you.

**The app starts empty, and empty means empty.** `factoryDefaults()` sets capital and
goal to 0 and deploy, split and `proj.mult` to `null`. Those five figures decide the
outcome more than any single trade does, and they used to arrive pre-filled — 2500 /
50% / 5.00x / 33% / 1m — reading as the owner's own numbers before any decision had
been made. Worse, "reset everything" restored them, so clearing never actually
cleared. **Do not reintroduce a starting value for any of the five.** An unset number
renders as an empty box, never as a zero standing in for a choice.

`S.planSet` gates this. The plan band at the top sets all five in one place and
collapses to a summary once saved; logging a trade is locked until it is. That lock is
load-bearing — it is why no render path can meet a `null` plan number.

**The plan band is a second way in, not a second copy.** It writes to the same
`S.capital` / `S.deploy` / `S.split` / `S.proj.mult` / `S.goal` as the controls further
down, and `renderPlan()` keeps the duplicated boxes in step. The per-trade selects on
the form still win for their own trade; the band only supplies the default they start
from, which is what keeps history stable when the plan changes.

**Demo mode owns the worked example.** `demoState()` carries the recommended
configuration and four trades that are deliberately not a clean sweep — one wipe and
one profitable miss, giving a 50% hit rate against a 75% win rate. It exists to show
the win-versus-hit distinction, not to flatter the strategy; keep it honest if you
touch it. `S.demo` shows the banner, and clearing it goes back to empty via the same
`loadState(factoryDefaults())` that reset uses.

**A separate "reserve pot" was considered and rejected.** Holding a reserve and
refilling after a wipe is just a lower deployment fraction wearing a disguise, and a
worse-behaved one: 33% reserve with an all-in pot risks 67% of capital per trade.
Simulated at a 70% hit rate — reserve-with-refill 26%, deploy-50% 99.5%. It also
introduces a discretionary "do I refill?" moment during a drawdown. Two pots, one
fixed fraction.

**Cash in and out is tracked separately from profit.** `invested` moves with
deposits and withdrawals so the growth percentage never counts money the owner paid
in as money they made.

**The projection no longer assumes every trade wins.** `ladderOdds()` runs a Monte
Carlo (deterministic LCG, no `Math.random`) from the current position and reports the
probability of reaching `S.goal`, alongside the same ladder staked all-in. It uses
the realised hit rate once there are 5+ trades with a target, and says plainly that
70% is an assumption until then. Keep that honesty — the odds are only ever as good
as the hit rate fed in.

**Deviation cost is measured in rungs, and only for early exits.** A planned win
multiplies the pot by `g = rungGrowth(f, m, k)`, so a trade that multiplied it by `a`
delivered `log(a)/log(g)` rungs and forfeited the rest of the one it owed —
`1 − log(a)/log(g)`, floored at zero. Losing trades are deliberately excluded: a loss
is the strategy working as designed, not a lapse, and folding it into the same number
would bury the thing being measured. `compute()` exposes `potBefore`, `deployActual`
and `growth` per trade, all derived from the replay rather than read from stored
fields, so entries written before those fields existed still measure correctly.

`tolDeploy` and `tolExit` are review tolerances, not plan figures — they do carry
starting values (2 points, 80% of target), because they are editable in the section
that uses them and the app states what they mean against the current plan. That is the
distinction: a default is acceptable when it is on screen next to its effect.

**Costs apply forward, never backward.** `S.fric` models the spread, slippage at size,
flat commission and tax. It is off and unset until switched on. A logged trade is real
cash in and real cash out, so the spread paid is already inside `start` and `finish` —
subtracting it again would understate the owner's own record and corrupt the hit rate
the whole plan is judged on. **`compute()` and `splitClose()` must stay untouched by
this.**

With frictions on, `S.proj.mult` is the **gross** move being hunted and `goalMult()`
returns what it nets. That works because `goalMult()` was already the single choke
point every forward surface reads from — including the target stamped onto a trade in
`saveTrade()`, so hit testing keeps comparing net against net without special-casing.
Net is resolved **per stake**, not once per plan: `projection()`, `goalPath()` and
`ladderOdds()` each call `netMult(gross, stake)` with their own row's stake, so the same
gross move buys less as the ladder grows. That degradation is the reason to model size
at all — don't collapse it back to a single plan-level figure.

Tax is a **liability, never a deduction**. `taxLedger()` buckets realised gains by UK
tax year (6 April), nets losses within a year, applies the allowance, and the result is
shown as money set aside from the safe haven. Taking it out of the pot instead would
change the rung count without saying so. Carry-forward between years is not modelled and
the interface says so. The rate comes from the user: the app cannot know their band, or
whether HMRC would treat this as capital gains at all rather than trading income.

**Saved plans live outside `S`.** `SAVES_KEY` (`tradingTracker.saves.v1`) holds a named
set. Deliberately its own key, because "Reset everything" replaces `S` wholesale and a
reset that also swallowed every saved plan would be a trap rather than a reset. Opening
one floors `S` to `factoryDefaults()` before `adopt()`, so nothing the save omits
survives from what was on screen.

## Gotchas

**`isFinite(null)` is `true`.** `null` coerces to `0`, so a plain `isFinite()` check
waves an unset plan number straight through and then throws on `.toFixed`. That is
exactly how the first pass at the empty state broke `renderSummaries`. Anything that
can be "not chosen yet" must be tested with `isNum()`, which checks `typeof` first.

**A destructive confirm has to outlive being read.** Both `resetBtn` and `wipeBtn` arm
on first click and fire on the second, through the shared `armConfirm()` helper. The
window was four seconds and changed only the label — long enough to start reading
"Sure?", not long enough to decide, so a working button was reported as a dead one.
Twelve seconds, and the armed state fills the button rather than just relabelling it.

**Never `localStorage.clear()`.** The live site shares an origin with unrelated
projects; `clear()` would take theirs too. `store()`/`fetchStore()` touch one key.

**Storage is scoped to the origin, and `file://` has no useful origin.** Opened from
disk, `trading-tracker_3.html` cannot see what was entered into `trading-tracker_1.html` —
the owner lost data this way. Always overwrite one canonical path; never hand back a
numbered copy.

Served over `https://` this reverses: `localStorage` is keyed to scheme + host + port and
the path plays no part, so every URL under the live site shares one store. Verified on the
deployed copy — a key written at `/Trading/trading-tracker.html` reads back at `/`.

**The live site is the canonical copy:** <https://acgroup21.github.io/Trading/>. Pages
serves `main` from the repo root; `index.html` is a redirect stub to `trading-tracker.html`,
not a second copy of the app. Local files and the live site are still separate origins, so
the JSON export remains the only way to carry data from a `file://` copy to the live one.

**The live site shares its origin with every other project on `acgroup21.github.io`.**
The tracker's `tradingTracker.v2` sits alongside unrelated keys from the other apps
published under that account. Namespacing keeps them apart, and `store()`/`fetchStore()`
only ever `setItem`/`getItem` that one key — there is no `localStorage.clear()` anywhere
in the file, so "Clear all entries" writes defaults over its own key and leaves the other
apps alone. **Keep it that way**; a `clear()` added here would now wipe unrelated projects.
Note that the browser's own "clear site data" for that host still takes all of them at once.

**Charts must not draw while their section is collapsed.** `drawEquity()` and
`drawMult()` bail out when the SVG has zero width, and a `toggle` listener redraws on
open. Without it a hidden chart renders against a zero-width box and comes out wrong.

**SVG text scales with the viewBox.** `setScale()` computes `K` from the rendered
width and `fs()` multiplies font sizes by it, so labels stay legible on a phone.
New text nodes need `fs()`, not a hardcoded size.

**The market dropdown caps at 600 options** and says how many it withheld. There are
1,114 markets; rendering all of them on every keystroke is wasteful.

**Typing in the market search snaps the category back to "All"** so you never search
a filtered subset and wonder why Bitcoin will not appear.

## Testing

Playwright against the bundled Chromium — there is no dev server, load the file
directly:

```js
chromium.launch({executablePath:'/opt/pw-browsers/chromium-1194/chrome-linux/chrome'})
page.goto('file:///absolute/path/trading-tracker.html')
```

Sections are `<details>` and collapsed by default; open them before reading text:

```js
await page.evaluate(() => document.querySelectorAll('details').forEach(d => d.open = true));
```

Always assert on `pageerror` and `console` errors, not just visible output. Check the
maths by hand for at least one full sequence — a clean render proves nothing about
whether the numbers are right.

## Style

- British English, sentence case, plain words. "How much of the pot goes in?" not
  "Deployment fraction".
- Money via `gbp()`, multiples via `xm()`, never raw `toFixed` in the UI.
- Colours come from the CSS custom properties at the top; both light and dark are
  defined, and dark is a chosen set of values, not an inversion.
- Every input the user types into gets a clear button (`data-clr`).
- The tool states its assumptions where the user will see them. It is a calculator,
  not advice, and it should not flatter the plan.
