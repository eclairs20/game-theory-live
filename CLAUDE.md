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
- `main/subs/<roundKey>__<encodeKey(studentId)>` — one per student per round:
  `{ roundKey, game, round, studentId, name, ts, guest?, value|choice|contrib }`
  where `roundKey = game + "-" + round`. `studentId` is the Google email (for signed-in
  students) or a random `s_…` id (for name-join). The path segment is `encodeKey()`d because
  Firebase keys can't contain `. # $ [ ] /`; the doc keeps the raw `studentId`. `guest:true`
  marks a submission whose id isn't on the roster.
- `main/roster` — array of `{ email, name }` for the enrolled class (optional). When present,
  the student join screen prefers Google sign-in; a Google email matching the roster joins
  under that name, a non-matching email joins as a flagged `guest`. Name-join is the fallback.
- The data layer is a thin adapter over the Firebase compat SDK; the app only uses
  `.ref().on()/.once()/.set()/.update()/.remove()`. Preserve these paths and shapes.

## Student identity (Google sign-in + roster)
- Firebase Auth (compat) provides "Continue with Google". `applyIdentity()` recomputes the
  effective `myId`/`myName`/`meGuest` on every render from `authUser` (Google) or the manual
  name (`manualId` + `gtl_name`). Sign-in is optional: if the auth SDK is missing or the
  Google provider/authorized-domain isn't set up in the Firebase console, the app falls back
  to name-join and the Google button errors gracefully.
- One-time console setup for Google: Authentication → Sign-in method → enable **Google**;
  Authentication → Settings → Authorized domains → add `eclairs20.github.io`.
- Instructor console has a **Class roster** editor (paste `email, name` lines) and a live
  **Attendance** card (submitted vs not-yet, by name; guests flagged). Public DB rules mean
  the roster is world-readable/writable — same soft-gate trust model as `passHash`.

## The three games
- `guess23` — Guess ½ of the average (the `frac` default is now 1/2; key kept as `guess23`).
  Analysis: histogram + level-k markers, target, winner, implied reasoning level.
- `pd` — "Give or Keep" giving game (a Prisoner's Dilemma; the PD term is instructor-only,
  never shown to students). Config `{ keep, give }` (default 2/3): Keep adds `keep` to your own
  earnings, Give adds `give` to your partner's. Choices are `keep`/`give`. Students are randomly
  but deterministically paired (`pairSubs`, seeded by round). Analysis: choice split, average
  earnings vs all-Keep/all-Give benchmarks, shaded Nash/best-for-pair matrix (shading + legend
  only in results, never on the submission screen), a "who played whom" table (instructor) and
  each student's own match (on reveal).
- `publicgoods` — Public Goods. Endowment E (20), return-per-token `mpcr` (0.5). Analysis:
  contribution histogram, free-riders, group earnings vs Nash & social optimum, efficiency.

## Roles & flow
- Student: open link → sign in with Google (or enter a name) → submit → sees own submission +
  a count. Aggregates are hidden until the instructor reveals.
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

## Security / Firebase rules
See `SECURITY.md`. Two rule sets ship in the repo: `firebase-rules.stopgap.json` (a
blast-radius fix, safe to publish anytime — no sign-in required) and
`firebase-rules.lockdown.json` (real auth: only signed-in Google users read/write,
students submit only as their own verified email, only allowlisted instructor emails
write `control`/`roster`). The app supports the lockdown via `INSTRUCTOR_EMAILS`
(allowlisted Google accounts get the console with no passcode) and `REQUIRE_GOOGLE`
(when `true`: student join is Google-only, instructor access is allowlist-only). Both
consts live at the top of `index.html`; rules are pasted into the Firebase console by
hand (not auto-deployed).
