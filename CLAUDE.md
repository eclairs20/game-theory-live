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
- Only external runtime dependencies: the Firebase **compat** SDK (gstatic CDN) and Google
  Fonts. Everything else is inline — keep it that way. (The `qrcode-generator` library, MIT,
  is **inlined** as a `<script>` for the class-join QR — a bundled copy, not a new CDN dep.)

## Data model (Firebase Realtime DB)
- `ROOM` = one class (course + year), from the `?class=<id>` URL param (default `"main"`).
  All of a class's roster, sessions and data live under `‹room›/…`. Share a per-class link
  (e.g. `…/?class=pgp-2026`). Sections are NOT separate rooms — a section is a field on each
  roster entry, so a student attending another section is still enrolled and combined/cross
  analysis works. Firebase rules are room-generic (`$room`, not `main`).
- **Session + section (a "meeting")** — `control.session` (S1, S2…) is bumped per class
  meeting; `control.section` (`"" | "A" | "B" | …`) marks which section is meeting *now* —
  when unset but the roster defines sections, `curMSection()` defaults to the first section
  (there is no "None" option once a roster has sections). Both
  fold into the record key so a section's meeting and a replay never collide:
  `roundKey = "S‹session›‹section›-‹game›-‹round›"` (e.g. `S1A-pd-1`, or `S1-pd-1` when no
  section is set). `curSession()`/`curMSection()`/`roundKey()` handle this. On results/attendance
  the instructor picks a **scope** (`sectionScope`/`effScope()`): Section A's meeting, B's, or
  **Both** combined — `analysisSubs()` filters `allSubs` to `session+game+round` within that scope.
  Every meeting's data is retained separately.
- **Courses** — one deployment serves many courses/years. `courses/<id> → {name, createdAt}` is a
  global registry (outside any room) powering the console's **Course picker**; each course is its
  own room (`?class=<id>`). Saving a roster registers its course and writes an enrollment index
  `enroll/<encodeKey(email)> → {<courseId>:true}`. A student on the bare link (no `?class=`) is
  auto-routed after Google sign-in: one enrolled course redirects in, several show a chooser, none
  falls back to guest on `main` (`maybeAutoRoute()`/`coursePicker()`).
- `main/control` — a single object: `{ activeGame, round, phase, passHash, config, createdAt }`
  - `activeGame`: `"guess23" | "pd" | "publicgoods" | null`
  - `phase`: `"waiting" | "open" | "revealed"`
  - `passHash`: djb2 hash of the instructor passcode (front-end gate only, not real security)
  - `config`: per-game parameter overrides on top of `DEFAULT_CONFIG`
- `‹room›/subs/<roundKey>__<encodeKey(studentId)>` — one per student per round:
  `{ roundKey, session, game, round, studentId, name, ts, guest?, sid?, area?, section?, msection?, value|choice|contrib }`.
  `msection` is the meeting section running at submit time (drives the A/B/Both analysis scope);
  `section` is the student's enrolled section from the roster.
  `studentId` is the Google email (for signed-in students) or a random `s_…` id (for name-join).
  The path segment is `encodeKey()`d (Firebase keys can't contain `. # $ [ ] /`); the doc keeps
  the raw `studentId`. `guest:true` marks a non-roster submission; `sid/area/section` are stamped
  from the roster at submit time (denormalized for export/analysis). Instructor **CSV export**
  (`exportCSV`) dumps every sub in the room across all sessions.
- `‹room›/roster` — array of `{ email, name, sid, area, section }` for the enrolled class
  (optional). Parser (`parseRoster`) is header-aware (ID/Name/Email/Area/Section in any order,
  tab/comma/2-space separated) and also accepts a plain `email, name` list. When present,
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

## The games
- `guess23` — Guess ½ of the average (the `frac` default is now 1/2; key kept as `guess23`).
  Analysis: histogram + level-k markers, target, winner, implied reasoning level.
- `pd` — "Give or Keep" giving game (a Prisoner's Dilemma; the PD term is instructor-only,
  never shown to students). Config `{ keep, give }` (default 2/3): Keep adds `keep` to your own
  earnings, Give adds `give` to your partner's. Choices are `keep`/`give`. Students are randomly
  but deterministically paired (`pairSubs`, seeded by round). Analysis: choice split, average
  earnings vs all-Keep/all-Give benchmarks, shaded Nash/best-for-pair matrix (shading + legend
  only in results, never on the submission screen), a "who played whom" table (instructor) and
  each student's own match (on reveal).
- `coord` — **Coordination** (G3) — teaches **focal points (Schelling)**. Four questions
  (`COORD_QS`) are answered **back-to-back in ONE submission** (NOT via rounds): Heads/Tails →
  arrange A,B,C → pick a square (2×2 with one odd-coloured focal square) → split $100 (claim a
  number; focal = half). Each answer **locks the instant it's chosen** and the next question
  appears (`studentCoord` shows the first unanswered question; `coordAnswer` merges the new field
  into the single sub doc and re-submits — students can't change earlier answers). Answers live in
  fields `c1..c4` on one doc; a 4-dot progress bar (`coordProgress`) and the four "coordinate
  without talking" rules (`coordRules`) stay visible; when all four are in, a locked summary shows.
  On reveal, `analyzeCoord` shows all four results together — one compact card per question
  (`coordQResult`) with the modal answer and % on the **focal** point (`coordFocalNote`). Just
  Open → students play all four → Reveal. No pairing; whole-class.
- `publicgoods` — Public Goods ("Maximize your marks") — now **G4**. Config `{ E, mult }` (default E=8 marks,
  mult=1.5). The **whole pot is multiplied by `mult` and split equally among the N players**, so
  each player receives `mult·total/N` (your own share of a contributed mark is `mult/N`, which
  shrinks as the class grows). Payoff = `(E − contribution) + mult·total/N`. Nash = keep everything
  (`E` each); social optimum = everyone contributes (`mult·E` each); efficiency = average
  contribution rate. NOT the fixed-MPCR model (per-head no longer scales with N). Analysis:
  contribution histogram, free-riders, avg earnings per player vs Nash & optimum, efficiency.

## Interactive lessons (no submissions)
- **Pareto optimality** (`paretoLesson`, local `lessonMode="pareto"` + `PZ` state) — an
  instructor-launched, full-screen **payoff-matrix checker** (a teaching visual, not a game;
  touches no Firebase). Launched from a gold-shaded **IL1** tile in the console's **Choose
  exercise** picker (under an "Interactive lessons" divider, below the G1–G3 game tiles).
  It steps through
  the four-step test on each 2×2 cell — examine → scan for complete improvements (`pDom`: y is
  ≥ for both and > for one) → label dominated → identify Pareto-optimal — with a **Check all**
  that circles the whole Pareto set, **editable payoffs**, presets in `PARETO_PRESETS` (the
  Give-or-Keep bridge where the one dominated cell IS the Nash outcome; "efficient ≠ fair"; a
  single-winner case), and a companion payoff-space mini-map (`paretoMiniMap`). Add future
  lessons the same way: a `lessonMode` value + a full-screen builder + a Lessons-section button.

## Roles & flow
- Student: open link → sign in with Google (or enter a name) → submit → sees own submission +
  a count. Aggregates are hidden until the instructor reveals.
- Instructor: click "Instructor" → passcode → console (pick game, open/close submissions,
  reveal, next round, clear round data, edit parameters). First run sets the passcode.
- **Collection view** (`presentView`, local `presentMode` flag) — a full-screen, class-facing
  screen that auto-opens when the instructor presses **Open submissions** (also via the round
  bar's **Present** button). It shows participation only — live count, progress vs the section's
  roster, and a **join QR** (`qrSVG`/`qrFor` of `location.href`) — and **never any result**, so
  the instructor can project it safely while the console preview would otherwise leak the
  answer. Close/Reopen/Reveal/Fullscreen/Exit controls live on it; Reveal exits present mode.

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
(when `true`: student join is Google-only, instructor access is allowlist-only, and the
app is **sign-in-first** — `boot()` defers the `control`/`subs`/`roster` listeners until
`onAuthStateChanged` fires with a user via `attachData()`, and `detachData()`s on sign-out,
so nothing is read or shown before authentication and the rules can require `.read: auth != null`).
Both consts live at the top of `index.html`; rules are pasted into the Firebase console by
hand (not auto-deployed).
