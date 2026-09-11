# CLAUDE.md

Guidance for Claude Code (or any future contributor) working on this repo.

## What this is

**Plate & Progress** — a single-page workout tracker for a 3-day machine-only
Push/Pull/Legs split, built as an installable PWA (Progressive Web App) for
Android. No backend, no build step, no package manager, no framework.

## Stack

- Plain HTML + CSS + vanilla JS, all in one file: `index.html`.
- `manifest.json` — PWA manifest (name, icons, display mode).
- `sw.js` — minimal service worker (cache-first, enables offline use and
  installability).
- `icon-192.png` / `icon-512.png` — app icons.
- Persistence: browser `localStorage`, single key `tracker-data` holding one
  JSON blob (see Data model below).
- Fonts: Google Fonts (`Oswald` for headings, `Inter` for body), loaded via
  `@import` in the `<style>` block. Requires internet on first load only;
  cached by the service worker after that.

There is no build/bundle step. Editing `index.html` directly and reloading
the page is the entire dev loop. Don't introduce a bundler, framework, or
package.json unless explicitly asked — the zero-dependency nature is
intentional (it's what lets this be wrapped into an APK via PWABuilder with
no build pipeline).

## Running it locally

Any static file server works, e.g.:
```
python3 -m http.server 8000
```
Then open `http://localhost:8000`. Service workers require either
`localhost` or HTTPS — opening `index.html` directly via `file://` will skip
SW registration but the app still functions (it just falls back to a normal
page load instead of the cached/offline path).

## Data model

Everything lives in one `localStorage` key, `tracker-data`, as JSON:

```js
{
  bodyweight: number | null,
  lastBackup: string | null,   // ISO date of last export, drives the backup nudge
  exercises: {
    "<exercise_id>": { weight: number | null, unit: "kg", fails: number, note?: string }
  },
  history: [
    { id, name, date (ISO string), weight, unit, reps: number[], result }
  ]
}
```

The app is **kg-only**. The `unit` fields are always `"kg"` — they are kept
in the schema only so old exports keep importing cleanly; don't add UI or
logic that branches on them. A history entry's `weight` is the weight
**actually lifted** that session (captured before the progression rule
adjusts the working weight).

The canonical exercise list (id, day, name, rep range, muscle-group size
category, set count) is hardcoded in the `DEFAULT_EXERCISES` array in
`index.html`. Icons live in `ICON_PATHS`, keyed by the same exercise id.
Starting-weight bodyweight ratios live in `BW_RATIO`, also keyed by id.

If you add/remove/rename an exercise, update `DEFAULT_EXERCISES`,
`ICON_PATHS`, and `BW_RATIO` together — they're joined by `id` and the app
assumes all three have matching keys.

## Core logic — do not change without explicit confirmation

The progression algorithm lives in the `log` button handler inside
`renderExerciseCard()`. Rules, in order:

1. All logged sets ≥ the exercise's `high` rep bound → **increase** weight by
   a fixed percentage keyed by muscle-group size (`INCREMENTS`: small 4%,
   large 5%, legs 7.5%), reset `fails` to 0.
2. All logged sets ≥ `low` but not all ≥ `high` → **hold** weight, reset
   `fails` to 0.
3. Any set < `low`:
   - First occurrence → **miss**, increment `fails`, weight unchanged.
   - Second occurrence in a row (`fails >= 2`) → **decrease** weight by 10%,
     reset `fails` to 0.

Weights move in `KG_STEP` (2.5 kg) increments — typical machine stacks.
Increases round **up** and are clamped to land at least one full step above
the current weight (so a weight sitting at 0 kg still moves); deloads round
**down** (`roundDownToStep`) but a weight at or below one step stays put
with a "rebuild the reps" message. Plain nearest-step rounding
(`roundToStep`) is only used for starting-weight suggestions.

An exercise with `bw: true` in `DEFAULT_EXERCISES` (back extension) is
**bodyweight**: no weight row or "Find start weight" button, history
entries record `weight: 0` (displayed as "BW", sparkline plots total reps
instead of load), and the increase/hold/miss feedback talks about reps and
tempo instead of moving weight.

Each card has a free-text machine-notes input (seat height, handle
position) stored per exercise in `note` — persistent settings, not
per-session data.

The rest timer between sets is driven by `REST_SECONDS` (keyed by
muscle-group size, like `INCREMENTS`); it (re)starts whenever a set's reps
are entered or stepped. Each log shows a 10-second **Undo** button that
removes the history entry and restores the pre-log weight and `fails`
counter.

This is a deliberate double-progression / novice linear-progression scheme
(see the original project's sourcing notes if available). Every session is
appended to `history` regardless of outcome. Changing these thresholds or
percentages is a product decision, not a refactor — confirm with the user
before altering them.

## Storage & save reliability

`saveState()` writes synchronously to `localStorage` but is wrapped as if
async (kept `async`/`await`-shaped for parity with a prior cloud-storage
version of this app). It retries once on failure and shows a dismissible
toast (`showToast`) if a save still fails, plus a persistent toast if
retries are exhausted. If you touch this function, preserve the
user-visible failure signal — silent save failures are the single worst
regression this app can have (a logged set that silently doesn't persist).

Export/Import (in the History tab) serialize/deserialize the exact same
JSON shape as `tracker-data` and are the user's manual backup path, since
`localStorage` doesn't sync across devices or browsers. Export prefers the
Web Share API (share sheet on Android) and falls back to a file download;
either path stamps `lastBackup`, and the History tab shows a nudge banner
when there are ≥5 sessions and no backup in 14 days. Keep the exported
JSON schema backward-compatible if you change the data model — old exports
should still import cleanly, or `loadState()`/import should upgrade them.

## UI conventions

- Colors, spacing, and type are all CSS custom properties at the top of the
  `<style>` block (`--bg`, `--surface`, `--accent`, etc.) — change the
  palette there, not by hardcoding new colors inline.
- Mobile-first, single column, max-width 480px, bottom tab bar (`push` /
  `pull` / `legs` / `history`). This is meant to be used one-handed,
  standing up, mid-workout — keep interactions to one or two taps.
- On load the app opens on the suggested day (`suggestDay()`): the day
  already trained today, else the next in the Push → Pull → Legs rotation.
- Exercises already logged today render as collapsed "done rows" (tap to
  re-expand). Expanding/collapsing swaps the single card node in place —
  never call a full `render()` for it, that wipes reps typed into other
  cards. Logging the last exercise of a day pops a session-summary sheet.
- All confirmations and info popups use the in-app bottom sheet
  (`showSheet(title, bodyHTML, buttons)`) — never `alert()`/`confirm()`,
  which render as jarring system dialogs in the APK wrapper.
- History shows an inline SVG sparkline of weight over time per exercise
  (`sparkline()`), no charting library — keep it that way.
- Icons are hand-authored inline SVG pictograms (`ICON_PATHS`), not photos —
  intentional, to avoid copyright issues with real equipment photos and to
  keep the app fully offline-capable. Keep new icons in the same style:
  stroke-only, `currentColor`/`var(--accent)`, viewBox `0 0 64 64`.

## Known constraints / history

- This app previously ran inside an AI chat artifact sandbox using a
  proprietary `window.storage` API. That has been fully removed and
  replaced with `localStorage` — if you ever see `window.storage` reappear
  in a diff, that's a regression from copying code out of that older
  version.
- No test suite exists. Manual verification: load the app, log a session on
  each tab, confirm the feedback message matches the progression rules
  above, reload the page, confirm state persisted, then use Export followed
  by Import to confirm round-tripping works.

## Things to check before deploying changes

- [ ] No `window.storage` calls reintroduced.
- [ ] `DEFAULT_EXERCISES`, `ICON_PATHS`, `BW_RATIO` keys still match 1:1.
- [ ] `manifest.json` icon paths still match actual files in the repo.
- [ ] Service worker `ASSETS` list still matches actual filenames if any
      file is renamed.
- [ ] Progression math changes are confirmed with the user first.