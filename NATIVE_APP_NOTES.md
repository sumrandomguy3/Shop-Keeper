# Shop Keeper — Going Native (Planning Notes)

*Written as a starting point for turning Shop Keeper into a real app store app.
You can hand this whole file to Claude Code when you begin — it explains what the
app is, what carries over, and a sensible order to do things in. No prior reading
required; it's written for someone new to coding.*

---

## What Shop Keeper is right now

A single web page (`index.html`) that behaves like an app. It's built with:

- **Plain (vanilla) JavaScript** — no frameworks, just the language itself.
- **localStorage** — the browser's small built-in storage. This is where all your
  jobs, photos, and settings live.
- **A service worker (`sw.js`)** — the bit that lets it work offline and be
  "installed" to a phone home screen.
- **GitHub Pages** — free hosting at the `sumrandomguy3/Shop-Keeper` repo.

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

- All your screens and layout (the HTML and CSS).
- All your calculations: board feet, estimates, markup, tax, deposits, balance.
- The job/status model, the cut list, the calendar view.
- The estimate share text (itemized + summary).
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
- **The native filesystem** (via a Capacitor plugin) — best for photos, saved as
  real image files instead of giant text strings inside the database.

**Your existing Export/Import backup feature is gold here.** It's how you'll move
your real data from the current web version into the new native version safely,
and it's a safety net if anything goes wrong during the switch. Don't remove it.

Current storage keys, for reference when migrating:
- `wj_v7` — jobs
- `wj_v7_photos` — photos
- `wj_settings` — settings
- `gcal_token` — the saved Google Calendar login

### 2. Camera

Right now photos come in through hidden file inputs (one with `capture` for the
camera, one for the photo library). Native apps use a **camera plugin** instead,
which gives a better experience and proper permission prompts. The good news:
your photos are already being compressed before saving, so that logic stays.

### 3. The service worker

`sw.js` exists to make a *website* work offline. A native app is already offline
by nature, so this file's job mostly goes away. You won't need the cache-version
bumping dance anymore either.

### 4. Google Calendar sign-in

The current Google login flow is built for websites. Native apps sign in a bit
differently (a native Google sign-in plugin). The calendar *syncing* logic stays;
just the "log in" step changes. Your OAuth client ID will likely need a new
entry for the mobile app in the Google Cloud console.

---

## Rough order of operations

A sensible sequence, roughly easiest-to-hardest:

1. **Install the tools.** Node.js, then Capacitor. Claude Code can walk you
   through this and set up the project skeleton.
2. **Get the app running in the wrapper unchanged.** Before changing anything,
   confirm the current app loads and works inside Capacitor on a phone simulator.
   This proves the foundation before you build on it.
3. **Swap storage to SQLite + filesystem.** The biggest and most valuable change.
   Do jobs/settings first, then photos. Test that data survives closing and
   reopening the app.
4. **Swap the camera to the native plugin.**
5. **Swap Google sign-in to the native plugin** (or skip calendar sync in v1 and
   add it back later — totally fine).
6. **Polish:** app icon, splash screen, permissions wording.
7. **Submit.** Apple and Google each have a review process and forms to fill out.

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

Wrap the existing app with Capacitor, move storage off localStorage onto SQLite +
the filesystem, swap camera and Google sign-in to native plugins, and submit —
Android first if the iPhone setup is a hurdle. Most of what you've built comes
along for the ride.
