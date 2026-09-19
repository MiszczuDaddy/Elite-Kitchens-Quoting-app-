# Elite Kitchens quoting app

Context and work order for Claude Code. This file loads automatically at the
start of every session on this repo — if you're a fresh Claude session
reading this, you don't need anything pasted to you, this is the full
picture.

---

## 1. What this is

A single-file web app, `index.html`, ~2,300 lines, no build step. Vanilla JS,
Firebase compat SDK for Google auth and Firestore, served from GitHub Pages at
`https://miszczudaddy.github.io/Elite-Kitchens-Quoting-app-/`.

It writes kitchen quotes and invoices for Elite Kitchens, a one-man bespoke
kitchen business in Balbriggan, Co. Dublin. Thomas is the owner and the only
user. This app is used on live jobs — a bug here means a wrong price goes to a
customer, so correctness beats cleverness every time.

**File layout:** one `<style>` block, then a `<div class="scr">` per screen
(dashboard, customers, addcust, quotes, builder, settings, invoices), then one
large `<script>`. Everything is global. There is no framework and no bundler;
do not introduce one.

**Data model.** Everything lives in a single Firestore document per user at
`users/{uid}`, mirrored into localStorage under `ek3_*` keys. Top-level
arrays/values: `customers`, `quotes`, `invoices`, `settings` (`ST`),
`quoteCounter`, `invoiceCounter`, `invoiceNums` (map, see §2 item 6 below).

A quote is roughly:

```js
{ id, num, cid, doors, drawers, topboxes, status, notes,
  rates:{gs, gl, cemux, blum},            // unit prices FROZEN at save time — see §2 item 4
  ess:  {on, ppd, tppd, drawer},          // drawer: 'none'|'cemux'|'blum'
  prem: {on, ppd, tppd, drawer, desc},
  pp:   {on, ppd, tppd, drawer, desc, extras:[]},
  wt:   {wtOn, wtP},
  glass:{sq, lq},
  extras: [{name, qty, price, unit}],
  incl: {sink, extractor, removal, electrical, plumbing},
  showExVat, created }
```

`ST` (settings) holds package/glass/worktop/drawer prices plus business
details, all with sensible defaults in `DEF_SETTINGS`:

```js
{ ess_ppd, prem_ppd, pp_ppd, lam_rate, gs_price, gl_price, extras:[...],
  dr_cemux, dr_blum,                                    // drawer box prices
  biz_phone, biz_email, biz_web, biz_addr, biz_bic, biz_iban, biz_vat }
```

**Pricing helpers** (module-level, used everywhere a price is calculated):
`ratesOf(q)` reads a quote's frozen `q.rates`, falling back to current `ST`
for quotes saved before `q.rates` existed. `drawerPrice(rates,key)` is the one
accessor for drawer-box pricing. `curRates` is the *builder's* working copy —
`newQuote()` sets it from current settings, `editQuote()` from the quote's own
frozen rates — and the builder's live preview (`recalc()`) reads `curRates`,
never `ST` directly. `genPDF`/`genInvoice`/`quoteAmount` each start with
`const R=ratesOf(q)`. Don't reintroduce a direct `ST.gs_price` read (or a
hardcoded `{none:0,cemux:60,blum:100}` map) into any pricing path — that's
the exact bug §2 item 4 fixed.

## 2. Business rules

- VAT is **13.5%** throughout (`const VAT`).
- Three packages: Essential, Premium, Premium Plus. A quote can offer any
  combination; the customer buys **one**. Shared items (worktops, glazed
  doors, extras) apply to whichever package is chosen.
- Pricing is per door + per top box + per drawer box + shared items.
- **Never add workmanship-warranty language to quotes.** Standing instruction
  from Thomas. The "6-month snagging" line (now in the dark terms panel on
  the quote PDF) is a separate thing and stays unless he says otherwise. A
  *material-limitation* caveat (e.g. "laminate worktops aren't covered
  against water damage" — added under the Worktops line, only when a
  worktop is quoted) is a different category and is fine; it excludes
  something from cover rather than promising anything about the work.
- Invoice numbers must be **unique and sequential** — Irish VAT requirement.
  Allocated via `allocInvoiceNum(qid, pkgKey)`, one number per quote+package,
  reused on reopen. Never derive an invoice number from `q.num`.
- Settings changes must **never** alter a quote that has already gone to a
  customer — see the `rates`/`ratesOf`/`curRates` machinery above. If you add
  a *new* per-unit price anywhere (a new material, a new add-on category),
  it needs the same treatment: snapshot it into `q.rates` at save time, add
  it to `ratesOf()`'s fallback, and never read `ST.<new_price>` directly from
  `genPDF`/`genInvoice`/`quoteAmount`.
- The quote PDF (`genPDF`) deliberately never prints door / top box / drawer
  counts, or any per-unit rate. Printing a count next to a total lets a
  customer divide one by the other and read your per-door price straight off
  the document. Quantities stay internal. Don't reintroduce them.
- The quote PDF's "Every option includes" section is split into two labelled
  groups — "In your kitchen" (paid items: sink, extractor, worktops, glazed
  doors, custom extras) and "Work included" (labour: installation, removal,
  electrical, plumbing) — so a €900 upgrade doesn't read with the same
  weight as a free inclusion. Keep new inclusion types in the correct group;
  when in doubt, custom extras go in the kitchen group.

## 3. How to test

There is no test suite, and the app must not regress. Load `index.html` in a
headless browser (Playwright is pre-installed in the Claude Code sandbox —
`NODE_PATH=/opt/node22/lib/node_modules node yourscript.js`, no `npm install`
needed), push objects straight into the `quotes` / `customers` globals, and
call functions directly.

Two different output captures are needed — this catches people out:

```js
// genPDF writes into a new window
let written = '';
window.open = () => ({ document: { write(h){ written = h }, close(){} } });

// genInvoice builds a Blob and opens a blob URL
let inv = '';
const RealBlob = window.Blob;
window.Blob = function(parts, o){ inv = String(parts[0]); return new RealBlob(parts, o); };
window.URL.createObjectURL = () => 'blob:stub';
window.URL.revokeObjectURL = () => {};
```

To check what a customer would actually *see* in the PDF (as opposed to
string-matching the raw HTML, which false-positives on CSS — e.g. the class
name `.render-pg` also appears in the stylesheet, and `font-size:17px` will
match a search for the number 17), load the captured HTML into a **second**
real page via `page.setContent(html)` and read `document.body.innerText` /
query the DOM. This is how the render-page-count and door/drawer-count-leak
checks were actually verified — do the same for any new visual claim.

Firebase will fail to load offline — ignore `firebase is not defined`, it is
not a real failure in this environment (the CDN is blocked in the sandbox).

**Always test three quote shapes.** Old-shape quotes are the most common
source of crashes:

1. A full quote with all three packages, drawers, top boxes, glass, worktop
   and extras.
2. One missing newer fields — no `glass`, no `wt`, no `topboxes`, no `rates`.
3. A bare `{id, num, cid, doors}`.

All three must produce a PDF **and** an invoice without throwing, and every
screen (`renderQuotes`, `renderDash`, `renderInvoices`, `renderCusts`,
`editQuote`) must render for each. If you touch pricing, also: save a quote,
change every price in Settings, and confirm the saved quote's total and its
invoice are byte-identical before/after (this is the regression that matters
most — see §2).

Also check for duplicate element ids after any HTML change — one of the bugs
below was exactly that:

```bash
grep -ao 'id="[a-zA-Z0-9_-]*"' index.html | sort | uniq -d
```

## 4. How to work and push

- **Check before you fix.** Several items below may already be done. Read the
  relevant function first; if the fix is present, say so and move on. Do not
  reapply it differently.
- One branch per group of related items, e.g. `fix/mobile-layout`.
- Small commits, each with a message saying what broke and what changed.
- Verify by running the app, not by reading the diff.
- Push the branch and open a PR. Do **not** merge to `main` — Thomas reviews
  the diff first, because `main` is what the live app serves. (He'll ask you
  to merge explicitly once he's looked at it or the screenshot you send him.)
- If an item turns out to be a bad idea once you see the code, say so rather
  than implementing it.
- Send Thomas a screenshot of any visual change before/when asking him to
  review — render the captured PDF/invoice HTML in a real page and screenshot
  it, don't just describe it.

---

## Progress so far

Merged, in order (newest last). Read the linked commit message for full
detail — each one documents exactly what was verified.

| # | Branch | What it did |
|---|--------|-------------|
| 1 | `claude/keen-bohr-854bug` | Item 1, 2, and most of 3 below: fixed `genInvoice`'s `ReferenceError`, moved the render-image page into `genPDF`, guarded unguarded `q.<field>` access in `genPDF`/`genInvoice`/`editQuote`. **Did not** fix `renderInvoices`' `inv.total.toFixed` — that part of item 3 is still open, see below. |
| 2 | `fix/mobile-layout` | Item 14: sidebar → sticky top bar below 900px, `.row2`/`.row3`/builder grid/settings grid collapse to one column, builder summary becomes a sticky-bottom bar, tables scroll in their own box (`.tscroll`), inputs go to 16px to stop iOS zoom. |
| 3 | `fix/pricing-invoices-settings` | Items 4, 5, 6, 7, 8, and 23: froze `q.rates`, added `ratesOf`/`drawerPrice`/`curRates`, moved drawer prices and all business details into Settings, added `invoiceCounter`/`invoiceNums` so packages can't share a number, fixed the `id="s-gl"` duplicate, relabelled the dashboard "Quoted" tile. |
| 4 | `redesign/pdf-template` | Not in the original list below — a full visual redesign of the quote PDF (editorial layout, Faustina/IBM Plex Sans, package cards). Verbatim-replaced everything in `genPDF` between the rates/calculations block and the `window.open()` call. Introduced the "no door/topbox/drawer counts, no per-unit rates" rule now in §2. |
| 5 | `pdf-includes-split` | Not in the original list — split "Every option includes" into "In your kitchen" / "Work included", moved 6-month snagging into the terms panel. |
| 6 | `pdf-worktop-warranty-note` | Not in the original list — added the laminate-worktop water-damage caveat under the Worktops line, only shown when a worktop is quoted. |

**If you're picking this up fresh:** the pricing/rates/invoice-numbering
machinery (item 3 in the progress table) and the current `genPDF` HTML
structure (items 4–6) are both load-bearing for everything below. Read
`ratesOf`/`drawerPrice`/`curRates` and the current `genPDF` body before
touching pricing or the PDF template again.

---

# Work order

Priority order, as originally set. Items marked **DONE** below are finished
and merged — don't redo them; if you think one has regressed, verify first,
then fix and say what broke. Everything else is still open.

## 1. Invoice button throws — nothing happens at all

**DONE** (PR 1, above).

## 2. Render images never reach the quote PDF

**DONE** (PR 1, above).

## 3. Older quotes crash the PDF

**PARTIALLY DONE.** PR 1 guarded `genPDF`/`genInvoice`/`editQuote` — those no
longer crash on an old-shape quote. **Still open:** `renderInvoices()` reads
`inv.total.toFixed(2)` and `inv.paidAmount` unguarded (`index.html`, in the
invoice-list row template). An invoice record saved before some field
existed — or any `total`/`paidAmount` that ends up non-numeric — will throw
and break the whole Invoices screen, not just one row. Fix: `?.`/`Number(...)`
with fallbacks, same treatment as the rest of item 3 already got. Sweep the
file once more for any other unguarded `q.<field>.<sub>` or `inv.<field>`
while you're in there.

## 4. Settings changes retroactively re-price sent quotes

**DONE** (PR 3, above). See §1/§2 for how `q.rates`/`ratesOf`/`curRates` work
— read that before changing any pricing code.

## 5. Drawer box prices are hardcoded in four places

**DONE** (PR 3, above).

## 6. Two invoices can share one number

**DONE** (PR 3, above).

## 7. No VAT number on the invoice, and business details are hardcoded

**DONE** (PR 3, above).

## 8. "Quoted this month" counts the dearest package

**DONE** (PR 3, above) — label and comment fixed, maths unchanged (Thomas
wants the dearest-package figure kept).

## 9. Last-write-wins sync silently destroys work

**Still open.** Every save calls `.set()` on the whole `users/{uid}`
document. Two devices or two tabs; whichever saves last replaces everything,
no warning, no recovery.

**Fix.** Proper answer is one document per quote in a subcollection. Minimum
viable: store `updatedAt`, re-read it before writing, and refuse the save with
a clear warning if the remote copy is newer than the one loaded.

**Do items 9, 10 and 11 together** — they are all consequences of the
single-document design, and patching them separately means touching the same
code three times.

## 10. A large logo permanently kills cloud sync

**Still open.** The logo is base64 inside `settings` inside the one document.
`uploadLogo` allows 2MB; base64 inflates ~33%; Firestore caps a document at
1MB. Every subsequent write fails, with only a small grey "⚠️ Save error" in
the sidebar to show for it.

**Fix.** Downscale on a canvas to ~400px wide and cap around 200KB before
storing, or move the logo to Firebase Storage and keep only a URL. Either way,
make sync failures loud — a visible banner, not grey text.

## 11. Signing in wipes work started before sync landed

**Still open.** `onAuthStateChanged` fires a second or two after load, and
`fbLoad()` replaces `customers`/`quotes`/`invoices` wholesale.

**Fix.** Show a "connecting…" state and block saving until the first sync
completes; merge by id rather than replacing arrays.

## 12. Quote numbers can collide

**Still open.** `quoteCounter` falls back to `Math.max(0, ...)` which is 0 on
a browser that hasn't synced. A quote created in that window becomes #0001 on
top of an existing #0001. (Note: this is `quoteCounter`, the customer-facing
quote number — separate from `invoiceCounter`/`invoiceNums`, which item 6
already fixed with proper seeding-above-history + a reuse map. Quote numbers
have no such map yet.)

**Fix.** Covered by the item 11 guard. For real safety, allocate from
Firestore in a transaction.

## 13. Nothing can be exported, and there is no backup

**Still open.** There's a "Ready for Accountant" filter and a per-invoice
accountant toggle, but no way to send anything anywhere. The single Firestore
document is the only copy of every customer and quote.

**Fix.** Two buttons. **Export invoices to CSV** for a date range — number,
date, customer, net, VAT, gross, status — which is what the accountant wants.
And **Download backup**, dumping the whole dataset to a JSON file.

## 14. Unusable on a phone

**DONE** (PR 2, above).

## 15. Quote search can't find quote numbers

**Still open.** `renderQuotes` searches `String(q.id).slice(-4)` — the last
digits of an internal timestamp — instead of `q.num`. Typing `0088` returns
nothing. Search `String(q.num).padStart(4,'0')` instead.

## 16. No duplicate-quote button

**Still open.** Around 90% of jobs are full renovations with a similar shape,
and every quote starts from a blank form. Add a Duplicate action on each row:
copy the quote, clear `id`, `num` and customer, save as a new draft.

## 17. Nothing surfaces quotes going cold

**Still open.** Quotes are valid 30 days and there's a Follow-Up status, but
nothing shows a quote sitting at "Sent" for a fortnight. Add a dashboard
block: status Sent, created more than 10 days ago, showing days elapsed with
call and email buttons. Most likely item on this list to actually earn money.

## 18. Leaving the builder loses everything without asking

**Still open.** `builder-back-btn` goes straight to `goTo('quotes')`. No
confirm, no draft. Track dirty state and confirm; better, autosave the
builder to localStorage and offer to restore.

## 19. Deleting leaves orphans

**Still open.** Deleting a quote leaves its invoices behind, and their View
button then does nothing at all because `genInvoice` returns silently when
the quote is missing. Deleting a customer leaves their quotes as "Unknown".
Warn on delete when dependents exist, and make `genInvoice` report a missing
quote.

## 20. PDF and invoice open in popups

**PARTIALLY DONE.** A popup-blocked check was added on both `window.open`
calls (in `genPDF` and `genInvoice`) — a blocked popup now alerts the user
instead of failing silently. **Still open:** the deeper fix. Popups are still
blocked by default on mobile, and the invoice's "Save to Invoices" button
still depends on `window.opener` across a blob URL, which some browsers null
out — when that happens the invoice never reaches the records. Render the
preview in a full-screen in-page iframe instead; that removes the popup and
the `window.opener` dependency together.

## 21. Unticked packages show a stale price

**Still open.** `recalc` only updates `live-prem` / `live-pp` when those
packages are ticked, so an old figure stays visible in the collapsed header.
Reset to €0 in the `else` branch.

## 22. Settings prices save themselves without being saved

**Still open.** The extras price inputs write straight into `ST.extras` on
change, so the next `saveAll()` from anywhere persists them even if Save
Settings was never pressed. Hold edits in a temp object and commit on Save.
(Note: this is specifically about the *extras* price list in Settings —
`saveSettings()` itself now handles the drawer/business-detail fields
correctly, added in PR 3.)

## 23. Duplicate element ids

**DONE** (PR 3, above) — `s-gl` was the bug; re-run the grep in §3 after any
HTML change, since it's cheap and this class of bug is easy to reintroduce.

## 24. Unescaped `innerHTML` throughout

**PARTIALLY DONE.** `genPDF` has its own local `esc()` helper (added as part
of the PDF redesign, PR 4) and uses it for the values it interpolates —
customer name, extras names, package descriptions. **Still open everywhere
else:** `esc()` is scoped inside `genPDF` only, not a shared top-level
helper. `genInvoice`, and the list renderers (`renderCusts`, `renderQuotes`,
`renderDash`, `renderInvoices`), still interpolate customer/extras names into
template literals and `innerHTML` unescaped. Low risk with self-entered data,
but a name containing a quote or angle bracket breaks a table row or an
input. If you pick this up: hoist `genPDF`'s `esc()` to a top-level helper
and use it everywhere user data is interpolated, rather than writing a second
copy.

---

## Not for Claude — for Thomas

- **Check the Firestore security rules.** If they're still Firebase's default
  test-mode rules, every customer name, phone, email and address is publicly
  readable, and test-mode rules expire and break sync. They should be
  `allow read, write: if request.auth != null && request.auth.uid == uid;`
- **Ask the accountant** whether the invoice needs the VAT registration
  number (field added in item 7 — `ST.biz_vat`, fill it in under Settings →
  Business Details), and confirm 13.5% is right for supply-and-fit.
