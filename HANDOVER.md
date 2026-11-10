# Flip My Bird: developer handover

## What it is

A tracker for Australians who switch electricity and gas retailers to collect sign-up bonuses.
It answers four questions:

1. **Who am I with, and when can I switch without losing a bonus?** A countdown to the last of the
   deal's stay rules.
2. **What have I earned, and what's still owed?** A savings total and a monthly chart.
3. **Who will take me back as a new customer?** Each retailer has a "can't return for N months" window.
4. **Is a new deal worth it?** Cost after bonuses compared with staying put, using the household's
   grid usage and the plan's rates.

The owner uses it on their phone. It is in daily use with real data, so treat the data format as
something to migrate, not replace.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app, about 1,300 lines: markup, CSS and one script. No build step, no dependencies (Google Fonts only). |
| `example-data.json` | A made-up backup with five deals covering every feature. Loads through Restore. |
| `README.md` | User-facing summary. |
| `HANDOVER.md` | This note. |

Run it by opening `index.html` in a browser. To test the example file's **Load the example file**
button, serve the folder over HTTP (`python3 -m http.server`), because `fetch` doesn't work on
`file://`.

## Where it runs today

It is published as a **claude.ai Artifact** (a hosted page). Inside claude.ai the page gets a
`window.claude` object with a document database. Anywhere else, `window.claude` is missing and the
app falls back to `localStorage`. Both paths go through the same small `store` interface (see
"Storage" below), so swapping in your own backend means writing one more `store`.

## Data model

One **deal** (called a "flip" in the code) per retailer sign-up:

```js
{
  id: "mw6b7zz4enqwzg",            // client-generated
  retailer: "Engie",
  fuel: "Electricity",              // "Electricity" | "Gas" | "Electricity + Gas"
  start: "2026-08-31",              // connection date, YYYY-MM-DD, local time
  end: "2026-11-11" | null,         // null = current or considering
  considering: false,               // true = a deal being compared, not switched to
  stayMonths: 3,                    // retailer's minimum stay
  noReturnMonths: 12,               // how long after leaving they won't pay a new-customer bonus
  rates: { supply: 2.53, peak: 35, off: 24, fit: 3 } | null,  // supply $/day, others c/kWh
  bonuses: [{
    id, amount: 170,
    type: "retailer" | "econnex" | "cashback" | "other",
    source: "TopCashback",          // display name; for "retailer"/"econnex" it is fixed text
    viaEconnex: true,               // cashback site that routes through Econnex (still paid by the site)
    status: "pending" | "tracked" | "paid",
    stayDays: 45,                   // this bonus's own stay rule, 0 = none
    expectedBy: "2027-02-28" | null,// payout date the site quotes; null = guessed (see below)
    paidOn: "2026-11-20" | null     // set when marked paid
  }],
  notes: "", createdAt: "ISO timestamp"
}
```

**Usage profile**, one per person: `{ peak, off, exp }`, average kWh per day bought at peak,
bought off-peak, and solar exported.

Older records may lack `type` or `viaEconnex`. `bonusType()` infers the type from `source`, so keep
that fallback or migrate the records.

## The rules that matter

These are the parts that are easy to get wrong. Each is a small pure function in `index.html`.

- **Stay rules** (`stayRules`, `freeDate`): a deal is free to leave on the **latest** of the
  retailer's `start + stayMonths` and every bonus's `start + stayDays`. Month maths clamps to the
  last day of the month (`addMonths`: 31 Jan + 1 month = 28/29 Feb).
- **Return window** (`returnDate`): `end + noReturnMonths`. Retailer names are normalised
  (`norm`) so "Momentum" and "Momentum Energy" match.
- **One current deal per fuel** (`planSave`, `clashes`): saving a current deal
  - ends any older current deal for an overlapping fuel on the new start date,
  - replaces a matching deal in "considering" so bonuses aren't counted twice,
  - refuses a second deal with the same retailer over the same dates,
  - refuses to end a deal that starts after the new one.

  These came from real bugs (duplicate deals, double-counted bonuses), so keep them.
- **Cost model** (`dailyCost`): `supply + (peak × peakKWh + off × offKWh − fit × exportKWh) / 100`
  dollars a day. "Over the stay" multiplies by the days to `freeDate`; "really costs you"
  subtracts all the deal's bonuses.
- **When a bonus lands** (`expectedDate`, `bonusEvents`): the stay rule is how long you must
  stay, not when the money arrives. Paid bonuses sit on `paidOn` (or the deal start). Unpaid ones
  sit on `expectedBy` if entered; otherwise a guess: start + 6 months for cashback and Econnex,
  the stay-rule date for retailer credits. A bonus is overdue 14 days after that date.
  Bonuses on "considering" deals count as planned.

## Storage

`store` has three async methods: `save(deal)`, `remove(id)`, `saveUsage(usage)`. Reads come from
live subscriptions that set `flips` / `usage` and call `render()`.

| Who | Where | Paths |
|---|---|---|
| Artifact owner | claude.ai database | `flips/<id>`, `profile/usage` (owner-only rules) |
| Other signed-in viewers who can write | their private subtree | `data/users/<uid>/app/flips/<id>`, `data/users/<uid>/usage` |
| Everyone else, or outside claude.ai | `localStorage` | `flip-my-bird.v1`, `flip-my-bird.v1.usage` |

If a write to the database is refused, `goLocalAfterRefusal()` copies the in-memory state to
`localStorage`, sets the `flip-my-bird.v1.local` flag and carries on. **Backup / Restore** moves data
between any of these as JSON (`{ app, version, flips, usage }`, the same shape as
`example-data.json`).

## Ranks and the "what's next" line

`RANKS` (in `index.html`) is the bird ladder, keyed on money **received** (paid bonuses only).
Each rank has a `kind` drawn by `bird(mood, kind)`: one SVG skeleton whose parts and colours come
from the `BIRDS` table (beak, crest and tail variants), plus a separate egg. The app logo and icons
use the `galah` kind.
`rankFor()` picks the rank; a rise since the last visit (stored in `localStorage` under
`flip-my-bird.v1.rankAt`, as the rank's dollar threshold) shows a toast and confetti. `nextStep()` builds the one-line suggestion at
the top of the page in priority order: overdue bonus, no current deal, free to flip, two weeks out,
a bonus still "pending", otherwise "sit tight". The mascot's mood follows the countdown
(asleep, awake under two weeks, flying when free).

## Screens

- **Overview** (`#overview`): savings totals, a monthly bar chart stacked by bonus type
  (faded = unpaid) with a running-total step line, the current deal card, and deals being considered.
- **Details** (`#details`): bonuses still to come with Paid buttons, usage inputs, the return
  windows, every flip, and backup.
- **Deal form** (a `<dialog>`): kind (current / considering / past), dates, stay rules, rates,
  three fixed bonus slots (retailer, Econnex, cashback site) plus extras, notes. Shows a live cost
  estimate and warnings.

Chart colours were checked for colour blindness in both themes. Light and dark themes come from
CSS custom properties on `:root`.

## Screen sizes

Checked from 280px wide to 1440px, portrait and landscape, on both tabs and in both dialogs:
no sideways scroll, nothing off-screen or clipped, tap targets at least 28px. Rules live under
"Screen sizes" in the stylesheet: block containers get `min-width: 0` so wide content (the table,
the chart) scrolls inside its own box; hero type and the mascot scale with `clamp()`; under 340px
form grids stack; from 900px the two tabs lay out in two columns.

## What's not done

- **No reminders.** Nothing notifies the user when a stay ends or a bonus is overdue. This is the
  most useful next feature for a native app.
- **No real accounts outside claude.ai.** People outside the owner's organisation fall back to
  browser storage.
- **Rates are flat.** No shoulder rate, time-of-use windows, demand charges or seasonal usage.
  Usage is one average the user edits by hand.
- **Offer terms are not fetched.** The "check the terms" links go to the cashback site, OzBargain
  and a Google search. Stay periods are typed in by the user.
- **No automated tests.** It was checked by hand in a headless browser. The date and money
  functions are pure and easy to unit-test first.
- **One file.** Fine at this size. Split it up if you move to a framework.

## If you're turning it into a phone app

1. Lift the pure functions (dates, `stayRules`, `planSave`, `dailyCost`, `bonusEvents`) into a
   module and add unit tests.
2. Keep the data model, and write a migration that fills in `type` and `viaEconnex`.
3. Import the owner's data through **Download backup** → your import.
4. Add local notifications for: stay ending in 7 days, stay ended, bonus overdue.
