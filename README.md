# Flip My Bird

A tiny tracker for Australians who switch energy retailers to collect sign-up bonuses
(ShopBack, TopCashback, Econnex, retailer credits, referrals).

It answers three questions:

1. **Who am I with, and when can I flip?** Each deal can have several stay rules: the energy
   company's (in months) and one per bonus (in days, e.g. Econnex via a cashback site). The countdown
   waits for whichever ends last. Each bonus links to its cashback site, OzBargain and a terms search.
2. **Which bonuses are still owed?** Tracks each cashback as *not tracked*, *tracked* or *paid*.
3. **Who can I go back to?** Shows when each retailer you've left will treat you as a
   new customer again (often 12 months).

When you tap **I've flipped**, the old deal gets an end date and you add the new one. The form
warns you if you're leaving before the minimum stay or signing back up with a retailer that's
still in its no-return window.

4. **What will this deal really cost?** Put in a deal's rates (supply, peak, off-peak, feed-in) and
   your average daily grid use. Each deal shows its cost per day, over the stay, and after bonuses.

5. **Is this new deal worth it?** Add a deal you're considering. It shows its cost after bonuses
   next to staying where you are. Tap "I've switched to this" when you do, and your old deal ends.
   You can only be with one company per fuel; the app ends the old one or flags a clash.

## Running it

It's one file with no build step. Open `index.html` in a browser.

- Opened as a file or from any web host, data is kept in that browser's `localStorage`.
  Use **Download backup** / **Copy backup** and **Restore** to move it between devices.
- Published as a claude.ai Artifact, data saves to the owner's account and syncs across devices.

## Example data

`example-data.json` is a made-up backup with five deals: four past and current flips covering
retailer, Econnex and cashback-site bonuses (one through Econnex), plus a deal being considered.
To try it, open **Details → Backup → Restore**, then **Load the example file** (or choose the file),
and press **Restore**. The example deals all have ids starting `example-`, so they're easy to spot
and delete from **Every flip**. For a look without saving anything, use **Show me an example** on
an empty app.

## Birds

Money you've actually received hatches you up a ladder of birds, and the bird on the front page
changes as you climb: Egg, Galah ($10), Dodo ($50), Bin Chicken ($250), Moa ($500), Thunderbird ($850),
Roc ($1,300), Firebird ($1,850), Simurgh ($2,500), Garuda ($3,200), Griffin ($4,000) and
Phoenix ($5,000). Each step up is bigger than the last. Each one sleeps mid-stay, wakes when you're two weeks out, and flies when
you're free to flip. "See all the birds" shows the whole ladder, with the ones you haven't
hatched greyed out.

## Install it like an app

Open https://energynt-dot.github.io/flip-my-bird/ in Chrome on Android, tap the **⋮** menu, then
**Add to Home screen** (or **Install app**). On iPhone, use Safari's **Share → Add to Home Screen**.
It then opens full-screen from its own icon and works with no connection. `manifest.webmanifest`,
`sw.js` and `icons/` provide this; they are also what a Play Store wrapper (PWABuilder / Trusted Web
Activity) needs.
