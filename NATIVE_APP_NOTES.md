# Shop Keeper — Going Native (Planning Notes)

*Written as a starting point for turning Shop Keeper into a real app store app.
You can hand this whole file to Claude Code when you begin — it explains what the
app is, what carries over, and a sensible order to do things in. No prior reading
required; it's written for someone new to coding.*

*Last refreshed: June 2026, app cache version **v19**. The app has grown a lot
since this file was first drafted — the section below catches it up.*

---

## What's new since this doc was first drafted

If you read an earlier version of this file, here's everything that's been added
or changed, so the rest of the document makes sense:

- **Payment ledger** — replaced the old single "deposit" field. Jobs now hold
  multiple dated payment entries (amount, method, note), and job cards show
  balance-due / paid pills. Legacy deposit data auto-migrates.
- **Follow-up reminders** — per-job toggle with a configurable timeframe, a
  "Follow-ups due" section on the home screen, and cleanup when a job is deleted
  or restored.
- **Client contacts** — phone, text, email, Messenger, Instagram, and other,
  each with tap-to-open deep links and handle normalization.
- **Finish recipes** — a per-job ordered list of finish steps (product, coats,
  application) plus notes. This is an *internal* shop record and never appears on
  anything a client sees.
- **Global species price sync** — editing a lumber price can offer to update the
  same species across active jobs. Archived jobs are deliberately left untouched
  so old quotes stay historically accurate.
- **Materials upgrades** — board-foot and by-unit quantity modes, plus a separate
  "misc materials" list, and a rounding override on the estimate total.
- **Business profile + logo** *(v18)* — name, phone, email, address, and an
  uploaded logo, set in Settings. Feeds the new PDF documents (below).
- **Branded PDF documents** *(v18)* — proper letterhead Estimate and Invoice PDFs
  with your logo, generated for printing / sharing. Client-safe: markup is folded
  invisibly into the numbers, and hours and the cut list are kept off the client
  copy. The plain-text share is still there too.
- **Job & invoice numbering** *(v19)* — each job gets a `YY-MMDD` number from its
  creation date (editable), and invoices are `jobNumber-01`, `-02`, etc.
  (editable, with a "Next #" button). The old global `INV-0001` counter is retired.
- **Calendar decision** — the plan for Google Calendar has changed; see the
  rewritten **Calendar** section under "What needs replacing." This is important:
  the old plan would have carried a publishing blocker into the store app.
- **Design / UX polish** — SVG nav icons, bottom-sheet modal animations, iOS
  safe-area insets, press feedback, square photo thumbnail grid, in-app confirm
  modals (replacing the browser's `confirm()`), status pills, and a calendar view
  with due-date urgency coloring.

---

## What Shop Keeper is right now

A single web page (`index.html`) that behaves like an app. It's built with:

- **Plain (vanilla) JavaScript** — no frameworks, just the language itself.
- **localStorage** — the browser's small built-in storage. This is where all your
  jobs, photos, settings, and logo live.
- **A service worker (`sw.js`)** — the bit that lets it work offline and be
  "installed" to a phone home screen.
- **GitHub Pages** — free hosting at the `sumrandomguy3/Shop-Keeper` repo.

Feature-wise it now covers: job management with status tracking and an archive,
a calendar view, board-foot and estimate math (markup, tax, rounding, payment
ledger, balance), materials and cut lists, photos, finish recipes, client
contacts, follow-up reminders, data backup (export/import), branded Estimate and
Invoice PDFs, and job/invoice numbering.

It's what's called a **PWA** (Progressive Web App): a website that can pretend to
be an app. "Going native" means turning it into a true app you download from the
Apple App Store or Google Play.

---

## The two paths

### Path A — Wrap it (Capacitor) ← recommended starting point

[Capacitor](https://capacitorjs.com) takes your existing web app and puts a thin
native "shell" around it, producing a real installable app for both iPhone and
Android. **You keep almost all your current code.** Your screens, your estimate
math, your buttons — all of it comes along.

- **Pros:** fastest path, reuses what you've built, one codebase for both phones.
- **Cons:** it's still your web page running inside the app, so it won't feel
  *quite* as snappy as a fully native app. Apple sometimes rejects apps that are
  "just a wrapped website" — but apps with real device features (camera, offline
  data, calendar) like yours usually pass fine.

### Path B — Rebuild it (React Native or Flutter)

These build genuinely native screens from scratch. The app feels and performs
fully native, but you'd **rebuild the entire visual layer**. Your *logic* (the
board-foot formula, the estimate calculations, the job status rules) ports over
easily; the part that draws the screens does not.

**Suggestion:** start with Path A. If you outgrow it, the logic you carry over
makes Path B easier later. Don't commit to the harder path before you need it.

---

## What carries over as-is (the good news)

With Capacitor, most of the app moves with little or no change:

- All your screens and layout (the HTML and CSS), including the recent UI polish.
- All your calculations: board feet, estimates, markup, tax, rounding, the
  payment ledger, and balance.
- The job/status model, the archive, the cut list, and the calendar view.
- The estimate share text (itemized + summary) and the **PDF document builder**
  itself (the HTML/CSS that lays out the letterhead). Only the *trigger* that
  turns it into a printed/shared PDF needs attention — see below.
- Client contacts, finish recipes, follow-up reminder logic, and job/invoice
  numbering.
- The general structure of the photo and camera features.

---

## What needs replacing (the important part)

### 1. Storage — the big one

**localStorage is not safe for a real app.** On a phone, the operating system can
quietly wipe a web app's storage when it needs space. That's the exact "photos
didn't save / data disappeared" problem you've been fighting — and on native it
can get worse, not better.

The fix is to move your data into proper on-device storage:

- **SQLite** (via a Capacitor plugin) — a small built-in database. Best choice for
  jobs and settings.
- **The native filesystem** (via a Capacitor plugin) — best for photos and the
  logo, saved as real image files instead of giant text strings inside the
  database.

**Your existing Export/Import backup feature is gold here.** It's how you'll move
your real data from the current web version into the new native version safely,
and it's a safety net if anything goes wrong during the switch. Don't remove it.
Note: the backup file now also includes the logo, so a single export carries
everything.

Current storage keys, for reference when migrating:

- `wj_v7` — jobs. Each job object now also holds: `contacts[]`, a `payments[]`
  ledger, `finishSteps[]` / `finishNotes`, follow-up config, `jobNo`,
  `invoiceNo` / `invoiceSeq`, and `materials[]` / `miscMaterials[]` / `labor[]`.
- `wj_v7_photos` — photos, shaped as `{ jobId: [ {src, note, date}, ... ] }`.
- `wj_settings` — settings: defaults, lumber prices, and the **business profile**
  (name, phone, email, address).
- `wj_logo` — the uploaded business logo, stored as a PNG data URL. *(new in v18)*
- `gcal_token` — the saved Google Calendar login. **This goes away in the native
  build** — see the Calendar section.

### 2. Camera

Right now photos come in through hidden file inputs (one with `capture` for the
camera, one for the photo library). Native apps use a **camera plugin** instead,
which gives a better experience and proper permission prompts. The good news:
your photos are already being compressed before saving, so that logic stays.

### 3. The service worker

`sw.js` exists to make a *website* work offline. A native app is already offline
by nature, so this file's job mostly goes away. You won't need the cache-version
bumping dance (currently at v19) anymore either.

### 4. Calendar — swap it out, don't carry it over

**This is the most important change, and it reverses what an earlier draft of
this file said.** The current calendar feature signs into Google in the browser
and writes events through the Google Calendar API. That works for *you* today
only because the app is in Google's "Testing" mode, where just your own email is
allowed.

The problem: the calendar permissions Shop Keeper uses are what Google calls
**"sensitive" scopes**, and a public app using them must pass **Google's OAuth
verification** before anyone else can sign in — a privacy policy, a demo video, a
written justification, and a domain *you own and can verify* (the free
`github.io` address doesn't qualify, so you'd need to buy a domain), followed by a
review that takes weeks. Carrying the Google OAuth/API calendar into the store
app carries that whole blocker with it.

**The fix: in the native build, remove the Google OAuth + Calendar API path
entirely and use a native device-calendar plugin instead.** That writes events
straight to the phone's own calendar with a single one-time permission prompt —
no Google sign-in, no verification gate, works for every user. On most Android
phones the device calendar *is* the user's Google Calendar, so it still feels
like the sync you have now. It's also the same direction as today (the app
writing events out), so nothing about the experience is lost.

> **Rule of thumb for this step: swap, don't wrap.** If the native app keeps
> calling the Google Calendar API through OAuth, it inherits the exact same
> "only works for the test account" blocker. The calendar step is a *replacement*,
> not a port.

When you do this swap you can also delete the `gcal_token` storage and the
connect/disconnect controls in Settings — they're no longer needed.

*Lighter alternative if the plugin route is fussy:* generate a standard `.ics`
"Add to Calendar" file and hand it to the share sheet. It works on any calendar
(Google, Apple, Outlook), needs no accounts or verification, and is one-way /
on-demand — perfectly fine for due-date and follow-up reminders. Either approach
gets you out of the OAuth verification trap; the native plugin just feels more
automatic.

### 5. The branded PDFs (Estimate / Invoice)

The PDF *layout* (logo, line items, totals) is plain HTML and CSS and carries over
fine. What needs checking is the **trigger**: the web version builds the document
into a hidden frame and calls the browser's print dialog. Inside a Capacitor
webview that print call may not behave the same. The native fix is a print/share
plugin (or a render-to-PDF plugin) that takes the same HTML and produces a PDF the
device share sheet can save or send. Think of it like the camera and calendar:
the content stays, the device-level trigger gets a native replacement.

---

## Rough order of operations

A sensible sequence, roughly easiest-to-hardest:

1. **Install the tools.** Node.js, then Capacitor. Claude Code can walk you
   through this and set up the project skeleton.
2. **Get the app running in the wrapper unchanged.** Before changing anything,
   confirm the current app loads and works inside Capacitor on a phone simulator.
   This proves the foundation before you build on it.
3. **Swap storage to SQLite + filesystem.** The biggest and most valuable change.
   Do jobs/settings first, then photos and the logo. Test that data survives
   closing and reopening the app.
4. **Swap the camera to the native plugin.**
5. **Swap the calendar to a native device-calendar plugin** (remove the Google
   OAuth/API path — see the Calendar section). It's also completely fine to ship
   v1 with calendar left out and add it back here.
6. **Swap the PDF print/share trigger to a native plugin** so Estimate and Invoice
   PDFs generate and share on-device.
7. **Polish:** app icon, splash screen, permissions wording.
8. **Submit.** Apple and Google each have a review process and forms to fill out.

Tackle one numbered step at a time. Get each working before starting the next.

---

## Accounts and costs to expect

Verify current prices when you actually start — these change:

- **Apple Developer Program** — around $99/year (required to publish on iPhone).
- **Google Play Developer** — around a $25 one-time fee.
- A **Mac** is generally required to build and submit the iPhone version. Android
  can be built on any computer.

If the iPhone side is a hurdle, **launching on Android first is a fine plan** —
cheaper, no Mac needed, and you learn the whole process once.

---

## Tips for working with Claude Code (as a first-timer)

- **Start it by having it read the whole repo**, the way we did. Context first,
  changes second.
- **Work in small steps and test often.** "Get storage working for jobs" beats
  "convert the whole app" as a single request.
- **Commit (save a snapshot) after each working step**, so you can always go back
  if something breaks. Claude Code can do this for you.
- **It's okay to not understand every line.** Ask it to explain anything in plain
  terms — that's a normal and good way to learn.
- **Keep a backup export of your real data** before testing storage changes.

---

## One-line summary

Wrap the existing app with Capacitor; move storage off localStorage onto SQLite +
the filesystem; swap camera, calendar, and PDF printing to native plugins (and
specifically *replace* the Google OAuth calendar rather than carrying it over);
then submit — Android first if the iPhone setup is a hurdle. Most of what you've
built comes along for the ride.
