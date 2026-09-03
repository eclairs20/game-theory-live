# Game Theory Live — project context for Claude Code

## What this is
A live in-class instrument for MBA game-theory exercises. Students open one link on
their phones, submit choices, and the instructor reveals aggregated results + analysis
on a projector. Built for Prof. Sonia (IIM Lucknow); codebase managed by Ankit (eclairs20).

## Architecture (read this before editing)
- **`index.html` is the ENTIRE app** — HTML + CSS + JS all inline. No build step, no
  framework, no bundler. It must keep running as a plain static file.
- **Hosting:** GitHub Pages serves `index.html` from `main` at
  https://eclairs20.github.io/game-theory-live/. Deploy = `git push`; Pages rebuilds in ~1 min.
- **Backend:** Firebase Realtime Database (project `game-theory-live`, region
  asia-southeast1 / Singapore), under Prof. Sonia's Google account. The config lives in the
  `FIREBASE_CONFIG` const inside `index.html`. Rules are public read/write (classroom use).
  The Firebase web `apiKey` is public by design — it is NOT a secret; security is via rules.
- Only runtime dependencies: the Firebase **compat** SDK (gstatic CDN) and Google Fonts.
  Everything else is inline — keep it that way.

## Data model (Firebase Realtime DB)
- `ROOM` const (default `"main"`) namespaces all data. Change it for a fresh room/term.
- `main/control` — a single object: `{ activeGame, round, phase, passHash, config, createdAt }`
  - `activeGame`: `"guess23" | "pd" | "publicgoods" | null`
  - `phase`: `"waiting" | "open" | "revealed"`
  - `passHash`: djb2 hash of the instructor passcode (front-end gate only, not real security)
  - `config`: per-game parameter overrides on top of `DEFAULT_CONFIG`
- `main/subs/<roundKey>__<studentId>` — one per student per round:
  `{ roundKey, game, round, studentId, name, ts, value|choice|contrib }`
  where `roundKey = game + "-" + round`.
- The data layer is a thin adapter over the Firebase compat SDK; the app only uses
  `.ref().on()/.once()/.set()/.update()/.remove()`. Preserve these paths and shapes.

## The three games
- `guess23` — Guess ⅔ of the average. Analysis: histogram + level-k markers, ⅔ target,
  winner, implied reasoning level.
- `pd` — Prisoner's Dilemma (framed as two firms setting price). Payoffs R/T/P/S
  (default 3/5/1/0). Analysis: cooperation rate, shaded Nash/Pareto payoff matrix,
  expected payoff of cooperate vs defect given the class split.
- `publicgoods` — Public Goods. Endowment E (20), return-per-token `mpcr` (0.5). Analysis:
  contribution histogram, free-riders, group earnings vs Nash & social optimum, efficiency.

## Roles & flow
- Student: open link → enter name → submit → sees own submission + a count. Aggregates are
  hidden until the instructor reveals.
- Instructor: click "Instructor" → passcode → console (pick game, open/close submissions,
  reveal, next round, clear round data, edit parameters). First run sets the passcode.

## Conventions
- Theme-aware: light/dark via CSS tokens on `:root`, `:root[data-theme=...]`, and
  `prefers-color-scheme`. Keep BOTH themes working when you touch styles.
- Fonts: Fraunces (display), IBM Plex Sans (body), IBM Plex Mono (numbers/data).
- `localStorage` is used only for non-critical per-viewer state (student id, name, theme,
  role) and is wrapped in try/catch. Never put shared/authoritative data there.
- Charts are hand-drawn inline SVG (no chart library). Reuse the existing `barChart`,
  `histogram`, and `statTile` helpers so everything stays visually consistent.

## Deploy loop
1. Edit `index.html`.
2. Test locally: open `index.html` in a browser (it connects to the real Firebase), or
   `python -m http.server` and visit http://localhost:8000.
3. `git add index.html && git commit -m "..." && git push`.
4. Wait ~1 min for Pages, then hard-refresh (Ctrl+F5) — Pages/browser can cache the old file.

## Likely next tasks
- Add a game (sealed-bid auction, Cournot/Bertrand): extend `GAMES`, `DEFAULT_CONFIG`, add a
  `student<Game>()` view, an `analyze<Game>()` function, and a branch in `paramEditor()`.
- Optional hardening: tighten Firebase rules or add Firebase App Check to limit who can write.
- A per-round CSV export, or a persistent leaderboard across rounds.
