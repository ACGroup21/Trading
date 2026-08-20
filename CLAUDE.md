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

**The hit rate carries an error bar, and that is the point.** Everything downstream
rests on one number measured from a handful of trades. `wilson()` gives the 95%
interval — Wilson rather than the normal approximation, which collapses at small `n`
and near 0 or 1, exactly where a ten-trade sample lives. `oddsBand()` re-runs
`ladderOdds()` at both ends, so eight hits from twelve reports not "67%" but a true
rate of 39–86% and a chance of reaching goal between 6.9% and 100%. **Do not quietly
drop this to make the panel look tidier** — a sample that cannot separate success from
failure should say so. Shown only when the rate is measured; an override is an
assumption and assumptions have no error bar, and when the headline is still running on
the assumed 70% the band says which number is which.

**The page is three views, not one scroll.** `buildViews()` relocates existing blocks
at boot into set-up / log / review — three jobs done once, weekly and monthly, which had
been interleaved down 5,573px. Blocks are *moved*, not rewritten; to add one, put its id
in `VIEWS`. Tiles sit above the switcher because the summary is true in every view.
**`showView("review")` calls `render()` deliberately:** charts bail out at zero width, so
a panel that has been `display:none` draws nothing until it is on screen — the same trap
the `<details>` toggles handle.

**Entries is six columns with the rest one tap away.** `renderEntries()` emits a
`tr.main` followed by a hidden `tr.det` for every entry, across all three row types
(closed, cash, open). `toggleEntryRow()` opens the detail, and it ignores clicks whose
target is an `input`, `button`, `a` or `select` — without that, tapping the close-trade
box would toggle the row instead of focusing the field. **Keep `data-i`, `data-close`
and `data-doclose` on their elements wherever they move to:** every handler binds by
attribute, not by position, which is the only reason this restructure was safe.

**Anything claiming an assumption must name the number it actually used.** "What the
next trade needs" said "assuming you stake the [whole pot] you have free" while
`neededFor()` sized the trade at `cap * S.deploy`, so the multiplier on screen was
computed against half the money the sentence named and the two never reconciled. If you
touch `renderNeeds()` or `neededFor()`, check the arithmetic closes: for an equity
target, `cap - stake + finish + banked` must come back to the target, and the displayed
multiple must equal `finish / stake` for the stake named above it.

**Help lives in one table, not thirteen places.** `HELP` maps a section id to its
explainer, and `initHelp()` injects the `?` buttons at boot — to document a new section,
add a key. `guideHTML()` is the full guide behind the header button. Two traps, both hit
on the first pass:

- The `?` sits inside `<summary>` on most sections, so its handler **must**
  `preventDefault()` and `stopPropagation()` or opening the explainer also toggles the
  section.
- **Never inject into an element whose `textContent` gets rewritten.** `renderPlan()`
  rewrites the plan heading every render, which silently deleted the button — hence
  `<h2><span id="planH">`. Anything else that gains a `?` needs the same check.

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

## Phone

Checked at a real 375x812 with device emulation, not a narrowed desktop window — the
two are not the same and the narrowed window hid the bug below.

**Grid children need `min-width:0`.** They default to `min-width:auto` and refuse to
shrink below their own content, so `.two`'s Your record column rendered 398px wide in a
356px slot and pushed the document to 418px against a 375px viewport. Every grid in the
file (`.two`, `.mini`, `.tiles`, `.plangrid`, `.fgrid`, `.devtol`) sets it on its
children. **If a new grid appears, it needs the same line** — this is the failure mode
to check first when the page scrolls sideways.

**A collision sweep must filter to rendered elements only.** Content inside a *closed*
`<details>` still reports geometry here, so a naive overlap check reported two
collisions that do not exist. Filter on `offsetParent` plus open-ancestor checks.
Overflow checks alone miss wrap and overlap bugs, which is why both are run.

## Tablet

Swept 360–1180 in an iframe harness (one page, many widths) rather than resizing the
window repeatedly — the resize is flaky and the harness tests every breakpoint in one
pass. **Layout is sound across that whole range**: no horizontal overflow on any view at
any width, and `.two` flips one column to two at 861 as intended.

**Touch sizing keys on the pointer, not the width.** A tablet is touch at 768–1024, so a
width-only rule left it with 19px controls. The block is
`@media (max-width:640px), (pointer:coarse)`. Keep sizing in that block and *layout*
tweaks (full-width popover and guide) in the width-only one below it — a 1024px tablet
has room for a normal dialog and should not get the phone treatment.

Caveat for whoever tests next: this environment reports `pointer:fine` at tablet widths,
so the coarse branch could not be observed firing. It is verified present and
well-formed, not verified on real hardware.

## Desktop, and the two colour schemes

Swept 1280–2560: no overflow on any view at any width, zero collisions across 245
rendered elements, content capped at 1080px and centred. Desktop layout needs nothing.

**Light mode is the default and was unverified for a long time.** The dark palette sits
behind `prefers-color-scheme: dark`, so anyone whose OS is in light mode — including a
first-time visitor — gets the `:root` block. Measured, it failed WCAG AA in 92 places out
of 387, and **89 of those were `--muted` alone** at 3.2–3.5 where 4.5 is needed. Now
`#6b6a65`. **Check any muted-text change against `#f2f2ee`**, the darkest ground it sits
on, and measure both schemes — they have separate token blocks and fixing one proves
nothing about the other.

**Dark mode is fixed too** — it had 39 failures of its own. `--muted` is now `#949289`
(was 4.38) and `--critical` `#ff6b6b` (was 3.27, the worst in the app, and it lands on
losing multipliers, negative profit and rungs lost).

**Never hard-code `#fff` as text on a filled colour.** `--critical` is also a background:
the armed reset button puts text on it, and white on the brightened red reads 2.77, so
fixing the figures would have broken the button. Text sitting on a filled accent or
critical block uses `--on-accent` / `--on-critical`, which are white in light and
near-black in dark. Six rules had `color:#fff` and all six were wrong in one scheme.

The accent itself was deliberately **not** darkened to fix white-on-blue. `--series-1` is
the app's one strong colour and dulling it would flatten every chart line and border;
the blue stays and the text on it went near-black instead.

Both schemes measure 0 failures over 387 elements. Re-measure both after any token
change — they are separate blocks and a shared token like `--on-accent` moves both.
