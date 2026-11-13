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
- **Session + section (a "meeting")** — `control.session` (S1, S2…) identifies the class
  meeting and is **automatic** (see *Automatic session* below — there is no session stepper);
  `control.section` (`"" | "A" | "B" | …`, chosen by the instructor) marks which section is meeting *now* —
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
- `‹room›/roster` — array of `{ email, email2?, name, sid, area, section, audit?, drop? }` for the
  enrolled class (optional). Parser (`parseRoster`) is header-aware (ID/Name/Email/**Email2**/Area/
  Section/**Audit** in any order, tab/comma/2-space separated) and also accepts a plain `email, name`
  list. When present, the student join screen prefers Google sign-in; a Google email matching the
  roster (by **either** `email` or `email2`) joins under that name, a non-matching email joins as a
  flagged `guest`. Name-join is the fallback.
  - **`email2`** — an optional second address (e.g. a personal Gmail) that signs the **same** student
    in. `setRoster` indexes both addresses to one entry (`roster[email]` and `roster[email2]` → same
    object), so every existing `roster[<signed-in email>]` lookup matches either, and the student's
    past/future submissions stay linked whichever they use. `writeRoster` also indexes both in the
    global `enroll` map for auto-routing.
  - **`audit`** — marks a visitor/auditor who attends but isn't counted as enrolled. They still match
    the roster (name resolved, not a guest) and their subs are stamped `audit:true` (`meAudit` drives
    the stamp), but `enrolledRoster()` excludes them from the "X of N enrolled" denominators
    (attendance card, present view) and they're shown as **Audit** (not Enrolled/Guest) in the
    submitted table; CSV `enrolled` column is `yes`/`no`/`audit`/`drop`. Attendance lists auditors,
    dropped and guests separately ("Also submitted: N auditing, M dropped, K guests").
  - **`drop`** — marks a student who **dropped the course**. Mirrors `audit` exactly (parsed from a
    header-aware **Drop** column — `drop`/`yes`, or `"dropped"`/`"withdrawn"` in a Status column —
    subs stamped `drop:true` via `meDrop`, excluded from `enrolledRoster()`), but is labelled
    **Dropped** (red) in the submitted table and **"· dropped"** in roll call. The point: a dropped
    student **stays on the roster** so they're still recognised by name (not a guest) if they drop
    into an in-between session and can be ticked in roll call — they just don't count as enrolled.
    `drop` takes precedence over `audit` in labels/counts.
- `‹room›/schedule` — optional array of `{ date:"YYYY-MM-DD", session, section? }` mapping class
  dates to a meeting. `parseSchedule` accepts ISO or day-first (DD/MM/YYYY) dates, optional `S`
  prefix on the session, and an optional single-letter section (header row and junk lines skipped).
  The schedule is the **single source of truth for which session a date is** (see *Automatic session*).
  Edited via the Course config **Session schedule** card (`scheduleEditor`, paste like the roster; the
  textarea is pre-filled from the stored schedule, so in-app edits keep app-made changes).
- **Automatic session** — the instructor never sets the session:
  - **Games:** EVERY path that opens submissions (Open, Next round →, Reopen — console and present view)
    goes through `openMeeting(patch)`, which takes today's row for the running section
    (`todaySession(sec)` → `schedRowOn(date,sec)`: a row naming the section beats a shared row) and, if it
    differs from `control.session`, writes it with `round:1`. So a game can never be opened into a past
    session, and Reopen on a later day starts today's meeting (no late submissions to a past meeting).
    `openMeeting` is the ONLY writer of `control.session` (plus `undoSchedChoice` and the Run Undo `runUndo` restoring it).
  - **The console shows TODAY's meeting** (`liveSess()` = today's schedule row for the running section, else
    `control.session`; `liveRound()`; `todayStarted()` = students are already in it). Before today's first Open
    `control.session` still holds the last meeting, so the session label (`sessionTile`, read-only: "S5 · today ·
    starts when you open submissions"), `analysisSubs`, result headers, the present view and the attendance card
    all use `liveSess/liveRound` — round shows 1, and Reveal/Clear/round ± are locked until it starts.
  - **Stale round auto-close** (`maybeCloseStaleRound`, from `render()`, once per day per instructor console,
    only after BOTH control and the schedule have loaded — `scheduleLoaded`): a round still OPEN from an earlier
    session (control.session ≠ today's row, or today unscheduled) is closed, so no late submissions reach a past
    meeting. Never today's round; never in a course with no schedule.
  - **Freeze while submissions are open** (`isOpenNow()`, `lockIf(node,cond,reason)`, `OPEN_LOCK(what)`): game
    tiles, section switch, Next round →, Undo, Parameters (inputs disabled + note), schedule Save/Clear
    and Discard are dimmed and a tap toasts the reason; a 🔒 line above the round bar says so. The only ways
    forward are **Close submissions** and **Reveal**; **Hold** stays available (it pauses without losing
    anything). Normal flow is unchanged: Reveal → Next round works because submissions are closed by then.
  - **Unscheduled day** (`resolveTodaySession` → blocking `chooseModal`, `#chooseModal`; also a non-blocking
    console banner `unschedBanner` and an inline chooser in roll call): "which class is this?" —
    `unschedChoices(sec)` offers the **next scheduled session** ("S9 — moved from Thu 16 Oct", the usual case:
    a moved class whose row wasn't updated), the **most recent past session if nothing is recorded** for it
    (`sessRecorded`: roll-call ticks or subs; postponed class), a **New session S‹N+1›** when there's no
    future row (or no schedule at all — then that's the only option), and a minor **Extra class (S8.5)**.
    `applySchedChoice` writes it straight into the schedule (moving a row's date; a **shared** row is split so
    only this section moves) and sets `_schedUndo` → an **Undo** banner (`schedUndoBanner`, console + roll call)
    that works only while nothing is recorded under it. Only these two candidates are offered: a lecture with
    no roll call/game also looks "empty", so older sessions would be clutter.
  - **Extra classes** are stored as integer codes `1000+10·after+k` (`extraCode`; S8.5 = 1085, S8.6 = 1086)
    because the session is inside record keys, live-room names and roll-call keys and Firebase forbids "." in
    keys. `sessLabel(n)` renders "S8.5", `sessOrd(n)` sorts chronologically (S8 < S8.5 < S9),
    `isExtraSess(n)`. `parseSchedule` reads "S8.5". **Always label sessions with `sessLabel`, never "S"+n**
    (keys stay "S"+n).
  - **Schedule save checks** (`scheduleSaveOK`): refuses a session on two dates for one section (two classes
    would share a record key and a replayed game would overwrite); warns (confirm) when a save changes the
    date of a session that already has records (e.g. re-pasting an older spreadsheet after an in-app move);
    keeps recorded extra-class rows that a re-paste dropped.
  - **Run / Review (instructor console, every game)** — a two-part switch at the top of the console
    (`consoleSwitch`, `consoleView` = `"run"|"review"`, remembered per device in `localStorage gtl_cview`).
    - **▶ Run class shows exactly ONE round**: the newest round of the selected game + section today — the only one
      that can still change. There is **no round −/+** (going back used to reopen a played round under the same
      record key, so a re-submission overwrote it); the round is a label with an **"Earlier rounds → Review"** link.
      The Analyze bar (`analyzeBar`) only picks today's scope (Section A / B / Both → `sectionScope`).
    - **A round locks (moves to Review) the moment Run points elsewhere**: **Next round →** (`runNextRound`; asks
      first if the round wasn't revealed), **another game** (tile) or **another section** — the last two confirm
      when the current round has data and land on that game/section's **next unused round** today (`freshRound(g,sec)`,
      which also counts discarded rounds, so an old record key is never reused). All go through `runMove(question,
      doIt)`, which leaves a local **"↶ Back to it"** Undo (`_runUndo`/`runUndoOn`/`runUndo` — restores game, section,
      round, session, phase, `ps`, `robot`, `config`) until the new round gets its first record (`runHasData`).
    - **Discard… (Run)** replaces Clear: `discardRound` marks `control.discarded[<roundKey>] = ts` and moves Run to a
      fresh round. A discarded round's records are **kept but not counted**: `isDone` returns false for them (unless
      `_ignoreDiscard`), so every result, count, leaderboard, debrief and hub view skips them; the CSV has a final
      `discarded` column. Locked while open and when the round has no data. No Firebase rules change (it's in `control`).
    - **📊 Review results** (`reviewConsole`, not full-screen) — **read-only**: nothing writes `control`'s game /
      round / phase. An **indigo** band (`--review`/`--review-wash`, both themes) says so; a gold "Live now …
      ▶ Back to Run" line shows while a round is open (or an Undo is pending). The exercise tiles become a **filter**
      (`reviewGame`, with "N recorded rounds"); the meeting list (`reviewMeetings` — every recorded session · round incl.
      today's and discarded ones, newest first) has per-section chips (+ Both; discarded struck through) → `histSel` →
      `historyPanel(h)`: the game's own analysis (`histData`/`histAnalysis`, a discarded section drawn as it was via
      `_ignoreDiscard`), who submitted (✕ Remove per record), **📽 Project this** (→ `reviewPresentView`: class-level
      results on the projector, `_presenting`, never names; locked for a round that is open right now) and **Manage
      data**: Discard / **Restore** (`setDiscarded`) and **Delete permanently…** (`clearMeeting`, double confirm) —
      single section only. Permanent deletion lives ONLY here.
  - `pairSubs` (G1 pairing) seeds from the meeting's own roundKey, so a past meeting re-pairs exactly as it
    did live (it used to use the live `curKey`).
- **Students on hold** — `control.hold` (boolean). When true it freezes the **live class** only: the
  Class tab shows a neutral "your instructor will start shortly" screen (checked at the top of
  `studentView`, overriding game/phase/reveal); instructors are unaffected. When the **hub is on**,
  hold does *not* freeze the dashboard — `studentRoot` still renders the Class/My page nav, so
  students can open **My page** (past results, what they submitted, practice) while held, and
  `curStudentTab` defaults to the hub while held (the live tab is frozen). Without the hub, hold
  still blanks the whole student screen. Toggled from the **Students** control in `sessionTile`
  ("⏸ Hold" / "On hold — Go live ▶"); the console shows a reminder note while held. `hold` is part of
  `viewKey()` so flipping it re-renders students immediately. Use it to freeze the class activity
  while setting up or testing.
- **Sandbox / testing rooms** — a room whose id matches `isSandbox()` (`sandbox`, `dev`, `test`, or
  `‹those›-…`) is a throwaway test room: a gold `#sandboxBar` banner shows for everyone in it
  (`syncSandbox()`), and `writeRoster` **skips** the global course/enrollment registration for it, so
  no real student is ever auto-routed into a test room. The console's **Testing sandbox** rail
  section opens `?class=sandbox` and copies its link to share with testers (who join as guests with a
  Google sign-in). Real students only ever see their own room, so building/playing in a sandbox is
  invisible to them.
- **Manual attendance (roll call)** — a present/absent register the instructor ticks by name,
  **separate** from who submitted a game. Stored in `control.attend[<meetingKey>][<encodeKey(email)>]`
  = timestamp (meetingKey = `"S<session><section>"`, e.g. `S1A`) — in `control` so it's
  instructor-writable / world-readable like the hub, **no Firebase rules change**. The instructor
  opens a full-screen page (`rollCall()`, local `rollMode`) from the **top-bar "📋 Take roll call"**
  button (`#rollBtn`, shown/wired by `syncRollBtn()` for an instructor with a roster): a section-wise,
  searchable tick list built from the roster, in the roster's **saved order** (`setAttend`/
  `setAttendMany` write `control/attend/…` directly; audit members are shown but excluded from the
  "X of N enrolled present" count via `enrolledRoster`). **Roll call is fully independent of the game:**
  it has its own local session `rollSess` (null = today's, from the schedule via `todaySession`) and never
  reads or writes `control.session` (it used to write it with `phase:"waiting"`, which closed the live game
  mid-class). Its header −/+ stepper only changes what you're viewing ("back to today" link); on an
  unscheduled day it shows the "which class is this?" chooser and hides the tick list until settled. The
  header shows the meeting's **date** from the session schedule (`schedDateFor`/`fmtSchedDate`,
  matched on session+section). A meeting's roll is "taken" once it has any entry; a listed student
  without an entry counts **absent**. Helpers: `attendMap`, `attendKeyFor`,
  `parseAttendKey`, `isPresent`. Each tick re-renders (attendance is in `control`), so `_rollScroll`
  preserves the list scroll position. A **View** toggle (`rollGrid`) switches between two layouts:
  - **This session** — the single-meeting tick list, a **responsive multi-column grid** (`.roll-list`,
    `auto-fill minmax(360px,1fr)` — up to 3 columns on a projector, 1 on a phone) that fills the full
    page width (same `maxWidth 1180px` as the grid view). Each row shows, right after the name, a
    **P/A history string** (`.roll-hist`, e.g. `PPAP` — green P / red A) across every *taken* session
    for that section (`attendSessions(sec)`), so you can see a student's attendance pattern at a glance.
  - **All sessions** — an **Excel-like grid** (`.att-grid`): students (roster order, name **+ PGP id**
    `sid`) × every regular session `1..N` plus extra classes, in date order (`gridSessions(sec)` over
    `knownSessions(sec)` — scheduled, taken or played; S8.5 sits between S8 and S9). Sticky
    header row + sticky name column. Tap a **cell** to toggle that student present/absent for that
    session (`setAttend`); tap a **session header** to mark — or clear — the whole column
    (`setAttendMany`). Its scroll (both axes) is preserved across the per-tick re-render via
    `_rollGT`/`_rollGL`. The roll-call page is `maxWidth 1180px` for both views.
  A **read-only "played" overlay** (`playedMap`/`playedIn`) marks students who *submitted a game* in a
  meeting — a faint **played** chip in the single-session list and a faint **·** in the grid (a cell
  that is ticked present shows **P** instead). It **never writes** to `control.attend`: the
  instructor's manual ticks remain the single source of truth. (`playedIn` is **session-scoped, not
  section-strict** — a student who attended/played another section's meeting still shows "played"
  against their enrolled-section roll, since `msection` stamps the meeting they played in; matched on
  either address, excludes bots.) Students see their **own** record on My page via
  `hubAttendance()` (gated by `hub.attendance`), matched on their roster primary **and** `email2`.
- **Student hub ("My page")** — a personal student dashboard, gated by the instructor per course
  via `control.hub = { on, results, practice, lessons, attendance, resultKeys, lessonKeys, practiceKeys }` (all default
  **off/empty**; a **Student page** card in the console controls them). When `hub.on`, students get a
  **Class / My page** tab bar (`studentNav`; `studentRoot`/`curStudentTab` decide the default — the
  live class when something's open, else the hub; `onCtrl` resets `studentTab` on any activity change
  so an opening game pulls them back). The hub (`studentHub`) shows: **What you submitted**
  (`hubMySubs` — `allSubs` filtered to the viewer, matched on **all** their roster addresses
  (primary + `email2` + current login), so a game a partner recorded under their primary email still
  shows when they sign in with `email2`; `proxy` subs are included); **Class results** and **Interactive lessons** are **explicit opt-in per item** —
  the instructor ticks exactly which meetings/lessons are visible, so there's no ambiguity:
  - **Class results** (`hubResults`) shows the meetings whose roundKey is ticked in
    `hub.resultKeys` (`{<roundKey>:true}`) to **every student** — no per-section/attendance filter, so
    someone who missed the class can still learn from a shared meeting. The only exclusion is the
    round that is **open for submissions right now** (`control.phase==="open"` and it matches the
    current `roundKey`) so its live tally can't leak — a finished round parked in `"waiting"` (e.g. a
    one-shot game like Coordination) still shows once ticked. Returns `null` (card hidden) when nothing
    is ticked. The console lists every played meeting from `playedMeetings()` (deduped roundKeys with
    `isDone` subs, across ALL submitters — never restricted to the instructor's own) as tick-chips,
    plus **Select all / Clear**. Both the console chips and the student result buttons order via
    `meetingCmp` (game G1…G6 → session → section → round, so every instance of a game groups
    together) and label via `meetingLabels` (round shown
    only when a game+session+section has more than one round in the list), so nothing looks shuffled
    and repeated rounds are told apart. `control.published` is still written by `doReveal` but no
    longer read for display.
  - **Interactive lessons** (`hubLessons`) shows only lessons ticked in `hub.lessonKeys`
    (`{<id>:true}`); available lessons live in the `HUB_LESSONS` registry and launch via
    `launchLesson(id)`; `lessonVisible(id)` gates a student launching one (also enforced in `render()`).
    Add a lesson = one `HUB_LESSONS` entry + a `launchLesson` case.
  - **Practice vs a bot** (`hubPractice`/`practiceMatch` → `pairMatch` with `practice:true`) plays
    locally and **writes nothing**; the `hub.practice` master toggle plus per-game `hub.practiceKeys`
    (`{<id>:true}`) — the instructor ticks which of the practice-capable games (`HUB_PRACTICE`, the two
    card games quatro/redblack) students may practise. `hubPractice` shows only ticked games and
    returns `null` when none are ticked.
  All hub state lives in `control` (instructor-writable, world-readable), so **no Firebase rules
  change** is needed. `parseRoundKey` inverts `roundKey`; the student analysis is `analysisBody(game,
  historicalList, false)`, reused from `analysisView`.
- The data layer is a thin adapter over the Firebase compat SDK; the app uses
  `.ref().on()/.once()/.set()/.update()/.remove()` plus **per-child subs listeners**. Preserve
  these paths and shapes.
- **Subs sync is incremental, not whole-node.** `attachData` listens to `subs` with
  `child_added`/`child_changed`/`child_removed` (`onSubChild`/`onSubGone`), maintaining `allSubs`
  one row at a time and coalescing re-renders (`scheduleSubsRender`, ~60ms). A `.on("value")` on
  the whole `subs` node re-sends **every** submission to **every** connected client on **every**
  write — O(students × submissions) database Load that saturated RTDB with a big class (≈90% Load
  at ~50 students). Child listeners send only the one changed row, turning that O(N²) per-write
  fan-out into O(N). `allSubs` semantics are unchanged for all downstream readers. Don't revert to
  a whole-node `subs` value listener.
  **That fixed the DATA load, not the redraws** — see the *Live-update rendering rule* under Conventions.
- **Version banner** (`APP_VERSION`, `maybePublishVersion`/`syncVersionBanner`, `control.appVersion`)
  — GitHub Pages/browsers cache `index.html`, so a returning student can run a stale copy. Each
  deploy **bumps `APP_VERSION`** (string-sortable: ISO date + zero-padded suffix, e.g.
  `2026-10-03.002`). An instructor loading a newer build writes it to `control.appVersion`
  (instructor-only; merges, keeps other fields); any client whose embedded `APP_VERSION` is older
  then shows a dismissible bottom banner with a **Refresh** button (`location.reload()`). It
  **never auto-reloads** (that would wipe an in-progress record-mode entry) and is per-viewer
  dismissible. The banner only helps for transitions *from this version onward* (older cached pages
  predate the banner code).

## Student identity (Google sign-in + roster)
- Firebase Auth (compat) provides "Continue with Google". `applyIdentity()` recomputes the
  effective `myId`/`myName`/`meGuest` on every render from `authUser` (Google) or the manual
  name (`manualId` + `gtl_name`). Sign-in is optional: if the auth SDK is missing or the
  Google provider/authorized-domain isn't set up in the Firebase console, the app falls back
  to name-join and the Google button errors gracefully.
- **Sign-in is POPUP ONLY** (`googleSignIn`). The old fallback to `signInWithRedirect` stranded phone users on
  `game-theory-live.firebaseapp.com` with "Unable to process request due to missing initial state" — iOS Safari/Chrome
  partition the storage the redirect needs, and apps' built-in browsers (WhatsApp, Gmail, Instagram…) block it. Now
  `auth/popup-blocked` shows how to allow pop-ups, a closed pop-up just toasts, and an in-app browser (`inAppBrowser()`:
  known app tokens, Android `; wv)`, or an iOS web view without the `Safari/` token) gets "Open this page in Safari or
  Chrome" + **Copy link** (`signInHelp`, under the Google button). `getRedirectResult` at boot only clears an old pending one.
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
- `publicgoods` — Public Goods ("Maximize your marks") — labelled **G20**. Config `{ E, mult }` (default E=8 marks,
  mult=1.5). The **whole pot is multiplied by `mult` and split equally among the N players**, so
  each player receives `mult·total/N` (your own share of a contributed mark is `mult/N`, which
  shrinks as the class grows). Payoff = `(E − contribution) + mult·total/N`. Nash = keep everything
  (`E` each); social optimum = everyone contributes (`mult·E` each); efficiency = average
  contribution rate. NOT the fixed-MPCR model (per-head no longer scales with N). Analysis:
  contribution histogram, free-riders, avg earnings per player vs Nash & optimum, efficiency.
- `quatro` — **Simultaneous Quatro Uno** (**G5**) and `redblack` — **Red vs Black** (**G6**) are
  **real-time two-player games** played through the portal (in class, students who are physically
  playing with card decks still pair up here to record each hand and get live scoring). `studentPair`
  routes to a **waiting room** (`pairLobby`) or a **live match** (`pairMatch`).
  - **Waiting room / matchmaking** lives in an ephemeral node `‹room›/live/‹mc›` (`mc` = the game's
    `roundKey`, e.g. `S1A-redblack-1`). On opening the game a student joins the **lobby**
    (`lobby/‹enc(id)›` = presence with a 20s heartbeat + `onDisconnect().remove()`; entries older
    than 60s are filtered out). Tap a classmate → a directed **invite** (`invite/‹enc(toId)›`); the
    invitee sees Accept/Decline. **Accept** creates a `pairs/‹pid›` record (`pid` =
    `enc(idA)~enc(idB)`, key-safe) with **colours assigned at random** (guaranteed one Red + one
    Black — the winner is label-invariant, so random is fine) and, for G5, **a distinct random suit
    per player** (`pickTwoSuits` → `pair.suits`, keyed by `enc(id)`; `cardFace` renders a suit char
    ♠♥♦♣ red/black), so each player's four cards look like a real suit and whose-is-whose is obvious.
    It also sets the `of/‹enc(id)›` → `pid`
    pointer for both. Each side watches `of/<me>`; when it points at a pid they load the pair and
    enter the match. Works for **guests** too — matching is by presence, not roster identity.
    **Double-claim guard:** `liveAccept` claims BOTH `of` pointers with Firebase **transactions**
    (commit only if the pointer is still free or already ours), so a partner can never be grabbed by
    two people at once — if the inviter was claimed first, the acceptor bails cleanly ("just paired
    with someone else"); if the acceptor was, it releases the inviter's claim. `liveInvite` keeps only
    ONE outstanding invite (clears the previous), so you can't have two people accept you at once.
    `guardDb` passes `transaction` through (a no-op that never commits in read-only view-as-student).
    **Every way of becoming "occupied" claims `of` by transaction** — not just `liveAccept`:
    the **practice bot** claims `of/<me>` = `BOT_OF` (`"bot"`) (`liveEnterBot` withdraws your open invite;
    `botSync`, run from `studentPair` each render, claims it, so `onMyPair` drops you from the lobby and the
    heartbeat/`liveTick` stops re-adding presence), and `liveExitBot` releases it (a reload mid-practice
    releases the stale claim too). A **real pair pointer always wins** over the bot (`botSync` drops the bot
    game). Record-mode `recPairWith` claims both pointers the same way, refusing a partner already in
    another pair or practising. Without this, a student playing the bot stayed "free" in the waiting room
    and could be paired underneath it, orphaning the partner. **A real match only resumes a saved sub
    played against THIS partner** (`mySubWith(partner)` in `adMatch`/`pairMatch`) — otherwise an earlier
    practice-bot game at the same roundKey preloaded as "already placed", the student never re-saved, and
    their record kept `partner:"bot:…"` so the real pair could never zip. The
    listeners attach via `ensureLive(mc)` and are torn down by `detachLive()` (which also removes
    your lobby presence) whenever `render()` sees you're no longer on an open pair game.
    **Waiting-room liveness:** the realtime `.on()` listeners get throttled/dropped on a backgrounded
    phone tab, so a classmate joining or an invite arriving wouldn't surface without a page refresh. A
    **5s heartbeat** (`liveTick`) re-asserts presence AND actively **re-polls** the lobby/invite/pair
    (`livePoll`, `.once()` reads) as a safety net; a `visibilitychange` hook re-syncs the instant the
    tab regains focus; and the lobby has a manual **↻ Refresh** button. All three update the globals and
    re-render (or enter the match if `of/<me>` now points at a pid) without a reload. **Redraw only on a
    visible change** (`lobbyViewSig` = present players + incoming invite, excluding heartbeat `ts`; `pairLobby`
    records it in `_lobbySig`, `lobbyRender` skips identical redraws) — redrawing on every heartbeat made the
    chip you'd just tapped (the invitee you're waiting on) blink; that chip is now highlighted (`.pair-name.sel`).
  - **Live match** (`pairMatch`): the pair is **locked** — no changing partner or colour mid-game;
    an explicit **Leave match** clears your sub + `of` pointer and marks the pair `ended` (the
    partner is offered "back to the waiting room"). Play is **strictly sequential** — only the
    current (first-unplayed) hand is tappable, no skipping. Moves still write to each student's own
    **sub** incrementally (`savePair`, `done` flips true on the last hand), so `mutualPairs` +
    `analyzeRedBlack`/`analyzeQuatro` are unchanged and now zip reliably (both subs carry the real
    partner id). After both players commit hand *i*, `handResult`/`redBlackWinner`/`quatroPlay`
    reveal **who won that hand** and a running **scoreboard** (`pairScore`/`scoreboard`) updates;
    a **history** list (`matchHandCard`) shows each scored hand with explicit **You / opponent**
    rows (winner's row highlighted); G5 piles render as **overlapping cards** (`cardStack`, top-left =
    top of pile), and the G5 builder previews the pile you're stacking the same way.
    `isDone(s)` (`s.done!==false`) still gates every
    "submitted" count (`liveCount`, `presentView`, `attendanceCard`, analyses filter
    `isDone(s) && !s.bot`). **`isDone` also counts a `done:false` match sub as finished when the match is evidently
    over** — its moves (`subMoves`: `seq` entries or `piles`) reach the announced `rounds` (G9, G11), or the partner's sub
    is done, names this player back and has the same number of moves (all lockstep games: G5/G6 live, G7, G8, G9, G11).
    The client only saves `done:true` when the "Match complete" screen draws, so a phone locked right after the last
    tap left that player "in progress" and their partner flagged as unpaired.
  - **Practice bot** — a "🤖 Practice bot" button in the lobby sets `d.bot` and drops you straight
    into a match against a locally-generated ~Nash opponent (`botRBSeq`/`botPiles`), revealed hand
    by hand as you play; on completion `saveBot()` writes a synthetic `"bot:<myId>"` opponent sub so
    the analysis sees a full pair. Bot docs are `bot:true`/`guest:true`, excluded from counts.
    `orphanWarning` skips subs whose `partner` is `"bot:…"` (a solo practice game isn't an unpaired student).
    **Both synthetic-partner writers (`saveBot` and `saveRecPartner`) stamp `email` = the signed-in
    recorder's address** — the lockdown rules reject any sub whose `email` ≠ the writer's, so without
    it these opponent subs silently fail to save and the pair never zips. The `email` field is only a
    write-rule stamp; identity/enrollment is always classified by `studentId` (CSV export too).
  - Cards are inline-SVG faces (`cardFace(rank,kind)` — `"red"/"black"` suited for G6, a hex colour
    per number for G5; `cardBtn` taps). Config `{ rounds }` (`paramEditor` "Rounds played"; default
    `quatro:5`, `redblack:20`). The `live` node needs read/write in the Firebase rules (both tiers
    include it: open under stopgap, any signed-in user under lockdown). **`clearRound` ("Clear")
    removes the meeting's `subs` AND its whole `live/<mc>` node** — in record mode the entered hands
    live under `live/.../pairs/.../rec` until Done, so clearing subs alone would leave a half-recorded
    match in place (the student's `of` pointer drops them straight back into it, and the console shows
    no submission because none was saved). Clearing the live node makes Clear a real reset.
  - **G6 has two play styles** (`config.mode`, chosen in `paramEditor`; **default `"record"`**):
    - **`"record"`** (`pairRecorded`) — the real-cards/Excel workflow: pairs play all `rounds` hands
      with physical Red/Black decks, then record them here. Because it's **asynchronous** (offline
      play, then one person enters), record mode does **NOT** use the live presence lobby — instead
      `recordSetup` shows a **partner picker**: tap a **roster classmate** — or an **instructor**
      account (`INSTRUCTOR_EMAILS`, tagged "instructor", so a prof can test the picker or partner an
      odd-one-out student; self always excluded) — via `recPairWith(...,solo=false)` (sets `of`
      pointers for both, so the partner verifies later), or type a **name** to self-record
      (`solo=true`). For a big class (>8 candidates) a **search box** filters the chips by name or
      email in place (no full re-render, capped at 12 with a "+N more" hint) so it isn't a wall of
      names. The recorder then marks who held Red (`rec.redId`) and enters each hand
      (editable `recGrid`, running Red/Black score) into `live/…/pairs/‹pid›/rec`; on Done writes their
      own sub. **On Done the recorder ALSO writes the partner's side** (`saveRecPartner`) — for every
      partner, roster or solo — so the pair is complete and shows on the instructor page immediately.
      That proxy sub is stamped **`proxy:true`**, and the instructor pairings label (`pairName`) tags
      such a player **"(unconfirmed)"** until they self-confirm. A roster partner's confirmation is an
      **optional hygiene step**, not required: they keep their `of` pointer and can open the game to
      **Confirm** (overwrites their sub without `proxy`, sets `rec.verifiedBy`) or correct — gated on
      `!rec.verifiedBy`, not on whether their sub exists. **Done writes BOTH sides in ONE atomic
      update** (`saveRecBoth` → a single `fb.ref(subs).update({<myKey>:myDoc, <partnerKey>:proxyDoc})`):
      Firebase applies it all-or-nothing, so a pair can never be left one-sided AND both colours come
      from the same `redId` in the same write (no colour drift). If the update fails, Done shows an
      error and does NOT mark the pair done — the student just taps Done again. (`recPartnerDoc` builds
      the proxy doc, shared by `saveRecBoth` and `saveRecPartner`.) The verifier's Confirm checks its
      `savePair`. Per-tap draft writes to the live `rec` node are **debounced** (~450ms, flushed on
      Done) so a 20-hand entry is a few writes, not 40. **Restart (`liveLeaveMatch`) in record mode
      clears BOTH sides** — the recorder's sub AND the partner's PROXY sub (plus their `of` pointer and
      the `rec` draft) — so a redo starts clean and can't inherit a stale, mismatched half-row; a
      partner's confirmed/own (non-proxy) sub is left untouched.
      Any sub whose named partner has no submission is surfaced to the instructor by `orphanWarning`
      in `analyzeRedBlack`/`analyzeQuatro` ("⚠ N unpaired — partner not recorded", with the reason). `saveRecPartner` stamps a roster partner's
      roster fields and `guest:false` (a solo typed-name partner stays `guest:true`). **Both subs are
      ordinary per-player `redblack` subs** (`mode:"record"`, `color`+`seq`, mutual `partner`), so
      `mutualPairs`/`analyzeRedBlack` zip them into the same joint 4×4 matrix — no analysis change.
      `recGrid` renders each recorded hand's two cards in sized `.mini-slot` wrappers so they don't
      overflow the cell. Recommended in-class mode.
    - **`"live"`** (`pairMatch`) — the real-time on-phones match described above.
  - `quatro` (G5) — 4-card duel (1<2<3<4): stack your four cards into a pile, reveal top cards, lower
    discarded / equal → both discarded / **1 vs 4 → both discarded**; empty your pile first and you
    lose. **Non-transitive** (1 beats 4), like RPS — no dominant ordering. `studentQuatro` records
    `cf.rounds` piles (your own only) via `orderingPicker(current,onChange)` (tap the number cards in
    stacking order; ↺ resets). Stored as field `piles` = `JSON.stringify(["1234","4321",…])`.
    `analyzeQuatro` shows ordering popularity (all piles), best/weakest ordering by **computed**
    win-rate over mutually-paired hands (`quatroPlay`), tie %, and the non-transitivity note. No
    rule-check (nothing is hand-entered to check).
  - `redblack` (G6) — one player Red, one Black (coin toss); cards **K, A(=1), 2, 3**. Red wins if
    both play K or both play **different** numbers; Black wins if exactly one plays K or both play the
    **same** number — structurally favours Black (~60%), so "coin toss for colour isn't fair".
    `studentRedBlack` = pick partner + colour, then **tap the card you played each hand** (fast
    auto-advancing rail + an editable strip of mini-cards). Stores the **full per-hand sequence**
    field `seq` = `"K,A,2,3,…"` plus `color` — so the joint (Red-card × Black-card) distribution is
    recoverable (marginals alone can't reconstruct it — this was the whole point of the original
    Excel macro). `analyzeRedBlack` shows each colour's marginal mix, the **class joint 4×4 heatmap
    beside the Nash independent-product matrix** (`jointMatrixCard`; K 40% / A·2·3 20% → 16/8/8/8…),
    the **observed** Red/Black win split (from real zipped hands, not a formula), and a
    `pairingsCard` ("who played whom"). CSV/`subValue` carry the full `seq`/`piles` + partner.
- `attackdefend` — **Colonel Blotto** (**G7**; internal key kept as `attackdefend`) — a **real-time
  two-player** zero-sum game teaching **mixed-strategy equilibrium** (randomise, don't out-think).
  Story: two roads lead to a town; the **attacker** has **2 divisions**, the **defender 3**. Each
  round both place **all** their units across the two roads; the attacker **breaches** the town if on
  **at least one road** they *strictly outnumber* the defender (a tied road → a **coin flip**). Payoff
  = attacker's breach probability; the matrix is `AD_PAYOFF` (rows = attacker split 0..2 `AD_ATT`,
  cols = defender split 0..3 `AD_DEF`) = `[[0,½,1,1],[1,½,½,1],[1,1,½,0]]`. **Value = ⅔** (`AD_VALUE`),
  attacker optimal **(⅓,⅓,⅓)** (`AD_ATT_OPT`), defender optimal **(⅙,⅓,⅓,⅙)** (`AD_DEF_OPT`).
  - **Roles are FIXED for the whole match** (no mid-game swap — that was confusing): assigned at
    random when the pair forms by **reusing `pairData.colors`** (set for every pair in `liveAccept`) —
    **red → attacker, black → defender**; against the practice bot it's a one-time random `d.role`.
    A match is `rounds` (default **10**) lockstep rounds; both players keep their role throughout.
  - **Strict lockstep** (`adMatch`): `k` = first round not yet resolved by BOTH sides. You place your
    units for round `k`, your sub saves (`done:false`), then you **wait** ("your move is locked in… waiting
    for <partner>") until the partner places theirs — only then does the round resolve and round `k+1`
    open. You can never be more than one unresolved round ahead of your partner.
    **One screen at a time:** when a round resolves it shows its result on its own — the winner callout
    bar + `adBattle` scene + a **"Continue to round N ▶"** button — and only once you tap Continue
    (local `d.ack` = last-resolved round index) does the next round's placement appear. So a previous
    round's result is never left sitting above the board you're placing on (`resultCard(i)` is the
    shared renderer; the match-complete screen shows the final round's result too). After the **final
    round the Restart button is GONE** (`else if(k<R)`), so a stray tap can't wipe a completed result.
    The two roads are named **Road A** and **Road B** everywhere (`AD_ROADS`); a tie names its road
    ("Tie on Road A/B", via `adTieRoad`). **Restart**
    (`liveLeaveMatch`) in a live pair match clears **both** sides — your sub AND the partner's own sub
    (+ `of` pointer, + the pair's `flips`) — so neither reloads stale moves or lingers as a phantom
    submission, and removing the partner's `of` drops them straight back to the waiting room.
  - Reuses the G5/G6 **lobby/pairing** machinery (`studentPair("attackdefend")` → `pairLobby` →
    `adMatch`), but is a **self-contained** function block (`adMatch`/`adBattle`/`adTroopIcon`/
    `adSoldier`/`adShield`/`adCoin`/`adScenario`/`adPlaceDesc`/`analyzeAttackDefend` + the `AD_*`/`ad*`
    helpers) that does **not** touch `pairMatch`, so the card games can't regress. **Live-only**
    (`wantLive`). The full problem statement (`adScenario` — "The situation": two roads, 2 attackers /
    3 defenders, more-commands-the-road, ties a coin toss, attackers win on either road vs defenders
    must hold both) shows in the lobby and at the start of a match (cursor `k===0`, so it survives a
    refresh straight into round 1).
  - Each player stores only their **own** per-round placement as a **self-describing token** in field
    `seq` (comma-joined): `"A0/A1/A2"` attacking, `"D0/D1/D2/D3"` defending (index via
    `adSplitIdx(att,road1count)`). A round resolves once both tokens exist and are one attacker + one
    defender; `adBreaches(attIdx,defIdx,seed)` decides it. **Ties are a deterministic coin flip**:
    `adSeedFloat(adFlipSeed(a,b,i,attIdx,defIdx))` (djb2 + an integer-avalanche finalizer) < 0.5. The
    seed is **sorted pair ids + round + BOTH placed split indices**, so the flip depends on what was
    actually played — a fresh, unpredictable coin each match rather than a fixed per-round constant
    that repeats (and so looks rigged) across a pair's matches — while staying identical on both phones
    **and** in the instructor analysis, so both always agree. Verified fair: ≈50/50 overall, a binomial
    per-pair distribution, every tie cell ~0.50 (a lopsided run like 6/7 in one match is just variance).
  - **Explicit tie coin-flip.** When both have placed and the cell is a **tie** (payoff ½), the round
    does NOT auto-resolve: both see a gold "it's a TIE" banner + a pending battle board (neutral town,
    "⚖ COIN FLIP PENDING") + a spinning coin (`adCoin`), and the **attacker** presses **🪙 Flip the
    coin** to resolve it (the defender waits; in a bot match the human presses, whichever side).
    `adBattle(…,pending=true)` draws the pre-flip board. Gating helpers: `cellTie(i)`, `isFlipped(i)`,
    `roundDone(i)` (= both placed AND (not a tie OR flipped)); the lockstep cursor `k` advances on
    `roundDone`. The flip marker is shared via the **pair node** `live/<mc>/pairs/<pid>/flips/<round>`
    (both watch `pairData`; `adFlip` writes it) for a real pair, or the local draft `d.flips` vs the
    bot. The flip only gates **when** the tie reveals — the outcome is still the deterministic
    `adBreaches` seed, so clients never desync, and the sub's `done`/analysis are unaffected.
  - UI (`adMatch`, all **inline styles**, no new CSS): a persistent role banner with a troop icon
    (`adTroopIcon` — red soldier / blue shield, size-pinned so unwrapped flex children don't stretch);
    a live `scoreboard`; an explicit per-round **winner callout bar** ("🔥 Round N: Attacker broke
    through — TOWN LOST" / "🛡 Defender held both roads — TOWN SAVED", plus "You won/lost this round");
    `adBattle(attIdx,defIdx,breach)` — a ~360×210 inline-SVG battle scene (red soldiers vs blue shields
    on the two roads, per-road "breaks through / holds / coin flip", a castle that **flames + "🔥 TOWN
    BREACHED"** or flies a **green flag + "🛡 TOWN HELD"**, SMIL `<animate>`). A road with **no forces
    on either side (0 vs 0)** is labelled "— no forces here —", **not** a tie: it's definitive (the
    outcome matrix already treats it as no breach); only an EQUAL non-zero count (2v2, 1v1) is a real
    coin-flip tie. Then **interactive placement**
    — two tappable road lanes where you add/remove your units (no multiple-choice buttons), a "forces"
    pool, and a **Send into battle** button enabled only once all units are placed; and win/loss
    history chips. **Practice bot** plays the complementary role from the optimal mix (`adPickOpt`);
    `saveBot` writes its synthetic opponent (`mode:"ad"`, `seq`). Commit is
    `savePair({partner,partnerName,seq,mode:"ad"},done)`. A **bot game is a fresh throwaway** — it
    **never resumes a persisted submission**: both `adMatch` and `pairMatch` set `cur = (bot||practice)
    ? null : mySub()`, and the lobby "Practice bot" button resets the draft's play-state. Otherwise a
    prior sub at this roundKey reloads as your moves "already placed", so the bot plays against your old
    submission and every round resolves at once (the reported solo-vs-bot bug).
  - `analyzeAttackDefend` (reveal) counts the whole-class **attacker mix** and **defender mix** from
    the token prefixes (no pairing needed), each shown as **two bar charts side by side on a shared
    scale** (`grid-even`): the class mix (red/blue) next to the **Nash (optimal) mix** (gold bars —
    attacker 33/33/33, defender 17/33/33/17), so the gap from equilibrium reads directly. Plus the
    **breach rate** from `mutualPairs` round-aligned (stat tiles:
    rounds, town breached %, town held %, **game value 67%**), a lesson note, and a `pairingsCard` +
    `orphanWarning`. `subValue` renders `[tokens] vs partner`; `paramEditor` exposes **Rounds played**.

- `adwar` — **Advertising Game (Infinite)** (**G8**) — an **indefinitely repeated Prisoner's Dilemma** teaching
  repeated games / the shadow of the future. Two rival firms choose **High** or **Low** advertising each
  round; stage payoffs `RP_PAY[mine][theirs]` = (High,High) 4,4 · (High,Low) 6,3 · (Low,High) 3,6 ·
  (Low,Low) 5,5 (T=6,R=5,P=4,S=3; High is dominant one-shot). Config `{ cont, min, cap }` (`paramEditor`,
  `rpCfg()`): **`cont`** = % chance the market continues after each round (default **50**), **`min`** = a
  **hidden** minimum number of rounds every pair plays (default **20**; no stop draw before it; clamped to
  ≤ cap), **`cap`** = a **hidden** max number of rounds (default **35**). **Students see neither** — the game is presented as
  infinitely repeated ("the same rival, round after round"); no probability, "last round" or dice wording
  appears anywhere on the student side, and an ending just reads "🏁 The game is over after round N". The
  chance is only revealed in the post-reveal analysis note. The param note shows the
  expected length `rpExpLen(p,cap,min)` (= min + p + p² + … + p^(cap−min); ≈21 at the defaults — and 50% sits exactly on the grim-trigger threshold) and whether grim trigger sustains Low
  (`rpThreshold()` = (T−R)/(T−P) = **½**).
  - Live pairs via the shared lobby (`studentPair("adwar")` → `pairLobby` → **`rpMatch`**, self-contained,
    like `adMatch`; live-only via `wantLive`). **Lockstep**: n = rounds both have played; you choose for
    round n+1 only once round n resolved (waiting note otherwise). After each round the **continuation
    draw** `rpEnds(seed,i,mine,theirs,p,cap,min)` is deterministic — `adSeedFloat` of sorted pair ids + `mc` +
    the pair's `ts` (bot: a per-game nonce) + round + both moves — so both phones agree; never before `min`, always at
    the cap. The **payoff matrix (`rpMatrix`) is on every screen** — above the High/Low buttons each round, on
    the waiting note, and in each round result (played cell outlined). One screen at a time: round result →
    **Continue to round N** (`d.ack`). When the match ends: "🏁 The game is over after round N" + totals,
    `done:true` saved once (`d._doneSaved`), Restart removed.
  - Each player's sub: `seq` = `"L,H,L,…"` (own moves), `mode:"rep"`, `cont`, `min`, `cap`, partner fields.
    `done:false` while playing. Match length = the pair's (equal) seq length — the analysis never
    re-draws. **Practice bot** = tit-for-tat (Low first, then copies your last move); `saveBot` writes its
    sequence at the end.
  - `analyzeAdWar`: tiles (matches + avg length vs expected, both-Low %, both-High %, avg profit/round vs
    4 and 5), % Low by round, % Low after rival Low vs after rival High, a strategy classification
    (`rpStrategy`: Always Low / Tit-for-tat / Grim trigger / Always High / Other), the stage matrix, a
    theory note (threshold ½ vs the `cont` used; TFT is NE for p ≥ ½ but SPNE only at exactly ½), `pairingsCard` with
    totals + sequences, and `orphanWarning`.

- `adfin` — **Advertising Game (Finite)** (**G9**) — the **finitely repeated** twin of G8, teaching
  **backward induction / end-game unravelling**. Same stage game, same `rpMatch` (called with `fin:true`;
  `studentPair` routes both `adwar` and `adfin` there), same lobby, bot (tit-for-tat) and lockstep. Config
  `{ rounds }` (default **6**, `rpFinRounds()`), **announced to students**: lobby/header "exactly N rounds",
  "Round k of N", the last one flagged "— the FINAL round", scoreboard "k / N"; the match ends after round N
  (no draw). Subs carry `mode:"rep", fin:true, rounds` (no `cont/min/cap`). `analyzeAdWar(list, true)`:
  Matches tile "N rounds each (announced)", the % Low by round chart captioned for the end-game, and a
  backward-induction lesson note (unique SPNE = High every round) with the class's Low rate before vs in
  the last round.

- `robot` — **Axelrod Tournament** (**G10**; was "Robot Tournament" — internal key kept as `robot`, and students
  still "build a robot"; Public Goods is now labelled **G20**) — an Axelrod-style
  tournament on the G8/G9 Advertising Game (`RP_PAY`; Low = cooperate). **Teams** build one robot each; when
  the instructor presses **⚔ Fight** every robot plays every other one. Config `{ capMin:5, joinMin:5, buildMin:10,
  teamMin:4, teamMax:6, teams:15, rounds:100, disc:5, house:true }` (`rbCfg()` → `nTeams` = number of teams, 0 = no limit;
  `min`/`max` = team size incl. captain).
  **Flow: captains → join → build → fight.** (1) **Captains** (`capMin`): anyone taps *Become captain* and names a
  team (`rbCreateTeam`, blocked once `nTeams` teams exist); captains can rename (`rbRename`) or step down; the
  instructor can remove a team (`rbRemoveTeam`). At the timer the console runs **`rbLockCaptains`**: keeps the first
  `nTeams` teams by creation time (later ones dropped, their captains become players) and, if nobody volunteered,
  appoints random captains from the students who **tapped** (see *Active-only placement*), auto names. (2) **Join**
  (`joinMin`): non-captains join or **switch** teams (`rbSwitch`, cap `teamMax` incl. captain, enforced by
  transaction) until it locks. At the timer **`rbFinalize`** places every **tapped**-but-unteamed student into the
  smallest team (so captain-only teams fill first; past the cap if needed — nobody is left out), then enforces the
  **minimum team size** with `rbBalance` (pure, in memory): while there are more teams than `floor(total/min)` the
  smallest short team is dissolved (newest first on ties; its captain joins another team as a player), then any team
  still under `min` is topped up with random non-captain players from the biggest teams. Min beats max when both
  can't hold (7 students, min 4 → one team of 7). Auto-made teams use `rbTeamCount(n)` = min(ceil(n/max),
  floor(n/min)), capped by `nTeams`. (3) **Build** and (4) **Fight** as below.
  - **Active-only placement** — only students who **tapped something in this meeting** are ever placed automatically
    (`rbMarkActive` writes `rp/<enc(id)>/act` = ts; called by the **🙋 I'm in — put me in a team** button `rbImIn`
    (captains + join steps, for anyone not in a team), and by create / join / switch / leave). A page merely left open
    — e.g. a student from the previous section, whose phone follows whatever is running now — heartbeats presence but is
    never dealt into a team. Meeting-scoped, so attending another section still works. The heartbeat uses `update` (not
    `set`) so it keeps `act`; `rbEnsure` reads it back so "✓ You're in" survives a reload (`_rbAct`). The console splits
    unteamed students into "will be placed" (tapped) and "page open but haven't tapped — won't be placed"; the Present
    view tells students to tap. Late arrivals (after the join lock) are never placed — they Join a team with space.
  - **Instructor fixes**: a **✕** on every member chip in the console (`rbRemoveMember`, any phase before the Fight —
    a removed captain hands captaincy to the next member, the last member removes the team, and their `act` is cleared);
    **↺ Reopen building +2 min** in the locked phase (`rbAdjust({buildEnd})` — every team can edit again).
  - **Phases** live in `control.robot = {rk, capEnd, capLocked, joinEnd, finalized, buildEnd, fought}` (rk must equal `curKey`;
    `rbPhase()` → idle / captains / caplock / join / forming / build / locked / fought — a legacy state with only
    `teamsEnd` reads as past captains; from **server time** `srvNow()` via
    `.info/serverTimeOffset`). `openMeeting` starts a new meeting's team timer; a 1 s ticker (`rbTick`)
    updates `[data-cd]` countdowns in place, re-renders on a phase change (`rbPhase()` is in `viewKey`),
    heartbeats student presence, and lets the instructor console run `rbLockCaptains` / `rbFinalize` when each timer
    is up. Console card `rbConsoleCard` (+1 min / Lock captains now / Lock teams now / Lock robots now / Fight, teams list with robots,
    click a member to make captain); the round bar's Reveal becomes **⚔ Fight!** (`robotFight`); the
    Present view shows `rbPresentBody` (phase, big timer, team names — never robot settings).
  - **Teams** in the meeting's live node: `live/<mc>/rt/<tid>` = `{name, captain, members:{enc(id):{id,name}},
    bot, auto?}`, `rm/<enc(id)>` = one-team-per-student pointer claimed by **transaction** (create/join),
    `rp/<enc(id)>` = presence `{id,name,ts}` (15 s heartbeat; read with `.once`, never listened to).
    Listener `rbEnsure`/`rbDetach` on `rt` only, plus a **4 s poll** (`rbPollTeams`, `.once`, redraw only if the
    JSON changed) and a `visibilitychange` re-read, so a dropped/throttled phone socket still shows new teams;
    while a student types a team name the list holds still (render guard) and catches up on blur (`rbNewName` keeps the text). Join = transaction on `members` (cap `teamMax`).
    **No solos** (see the flow above; if no team exists at all at join lock, the present students are dealt into
    `rbTeamCount(n)` random teams). Late students can join a team with space during build. Clear also removes `control.robot`. **↺ Reset tournament** (`rbReset`, on the console card in
    every phase, NOT frozen while open) deletes the meeting's teams/robots (live node), any robot subs and
    `control.robot`, so the card returns to "Start team formation" — the way out of a test run with no teams.
  - **Captain-only editing**: the captain's draft (`draftFor("robot-"+mc)`) is written to `rt/<tid>/bot`
    (debounced 300 ms); teammates see a live read-only builder. Captain hands over by tapping a teammate;
    **"Captain offline? Take over"** succeeds only if the captain's presence is >60 s old.
  - **Robot = Calm/Angry state machine** (`rbNorm`): `open` (≤3 opening moves), `calm` L/H/P (P = Low but High
    `probe`%), `trig` row/total/never with `n`, `angry` 1/2/3/5/"until"/"forever", `forgive`% (rolled at the
    start of each angry round). Engine `rbDecide`/`rbUpdate`/`rbPlay` (verified: TFT copies, grim, TF2T,
    grudge). Presets `RB_PRESETS` (Always Low/High, TFT, Grim, TF2T, Generous TFT, Grim+forgiveness, Prober,
    Grudge holder); plain-English `rbSentence`; family `rbFamily` (`RB_FAMS`). Builder (`rbBuilder`) is all
    tap-chips (no selects, survives live re-renders) + a labelled 10-round **test drive** ("practice only, not
    part of your score") vs Always Low/High/TFT showing each rival, the move strips, profit and NPV, plus the
    payoff table (`rpMatrix(null, note)`).
  - **Done / Edit** (`rbSetReady`): the captain's **✓ Done** writes `rt/<tid>/ready=true` (flushing the debounced
    robot save in the same update) and the builder goes read-only with **✎ Edit robot** to reopen it while the build
    timer runs. Teammates see "Locked in by your captain"; the console shows **Robots done X/N**, a ✓ per team and
    "All robots are locked in — you can Fight now"; Present-view team chips get a ✓. Fight skips its confirm when
    every team is done. The instructor still decides when to fight.
  - **Typing then tapping**: inputs catch up on blur via `rbBlurRedraw` (≈300 ms later, and not while a pointer is
    down) — an instant `render()` on blur replaced the button being pressed, so "Become captain" needed several
    taps. `render()` also records `_rbLastPh`, so the 1 s ticker redraws only when a timer flips the phase.
  - **Fight** snapshots one sub per member: `{game:"robot", team, teamName, captain, robot:<JSON>,
    teamMembers, defaultBot, house, rounds, disc, …}` — the field is **`robot`, never `bot`** (`s.bot`
    marks practice-bot subs and is filtered out everywhere). Unconfigured teams fight as Tit-for-tat.
  - **Scores are NPVs** (aligned with the class's NPV/discounting): `disc` = discount rate % per round (default
    **5**, with **100** rounds a match — 100 rounds at 5% capture >99% of an endless game's NPV, so it behaves like the
    infinite game; at 10% a nasty-heavy class could let Always High win; 0 = plain totals), `rbPlay(A,B,R,r,rng)` returns totals `pa/pb` and NPVs `na/nb` (round 1 undiscounted,
    NPV = π1 + π2/(1+r) + …), `rbAnnuity(R,r)` = NPV of 1 a round (benchmarks: 5·annuity = all-Low, 4·annuity =
    ad war). There is **no mistakes/noise** option any more (removed on purpose — not taught).
  - `analyzeRobot`: dedupe by team, add house robots (`RB_HOUSE`), seeded round-robin `rbTournament(ents,R,r,seed)`
    (`rbRng` mulberry32; randomised robots replayed 5×; cached; `npv` per match ranks, `avg` = undiscounted
    profit/round shown alongside, W–L–T by NPV), a "how the score works" note (formula + benchmarks), tiles,
    full-height leaderboard (own team highlighted; members instructor-only; each robot's sentence), NPV by type,
    type-vs-type NPV heatmap, **evolution chart** ("Evolution — what if more firms copied the winning strategies,
    generation after generation?"; `rbEvolution` replicator dynamics over families on NPV, `rbLineChart`, a "How to read it"
    line, a computed caption naming the winner, boom-and-bust exploiters and types that died out, and a caveat line), Axelrod lesson note + the **NPV test** (perpetuity: cooperate 5+5/r vs cheat 6+4/r → cheating
    pays only if r > 100%, i.e. δ < ½ — the same threshold as G8).

- `bayes` — **Bayesian Game** (**G11**) — incomplete information / **Bayesian Nash equilibrium**. Each round Nature
  secretly draws the game: **Odd-Even** (chance `pOdd`, default **25%**; Player 1 wins on an even total, Player 2 on an
  odd total — matching pennies) or the **Matching Game** (both score 1 if they show the same number — coordination).
  **Player 2 is told which game; Player 1 only knows the odds.** Both show 1 or 2. Config `{ pOdd:25, rounds:20 }`
  (`bgCfg()`); to compare probabilities the instructor plays another round with a different `pOdd` (each round's results
  are separate, Analyze bar). Rules wording: P1 "you win on an even total", P2 "you win on an odd total".
  - Live pairs via the shared lobby (`studentPair("bayes")` → `pairLobby` → **`bgMatch`**, self-contained like `rpMatch`;
    live-only via `wantLive`). Roles fixed per match: **red → Player 1, black → Player 2** (`pairData.colors`; fallback
    sorted id); vs the practice bot a one-time random `d.role`. Lockstep with one screen at a time (result → Continue).
    Nature's draw `bgDraw(seed,i,p)` = `adSeedFloat(sorted ids + mc + pair ts (bot: d.bgNonce) + round)` — never the
    moves — so both phones agree. P1 sees both tables + "Which game is it? You don't know"; P2 sees "This round you are
    playing ODD-EVEN (Type I) / the MATCHING GAME (Type II)", only that table and "🤫 Don't tell Player 1 which game it is".
    Tables are labelled as on the slides: "Type I · Odd-Even (OE) · p = 0.25" / "Type II · Matching (M) · p = 0.75". Payoffs `bgPay(type,a1,a2)` → [P1,P2].
  - Sub: `seq` "1,2,…" (own moves), `types` "M,O,…" (Nature's draws, so analysis needs no seed), `role` P1/P2,
    `mode:"bg"`, `pOdd`, `rounds`, partner fields. **Practice bot**: as P1 sticks to 2 when pOdd ≤ 50% else 50/50; as P2
    matches your usual number in Matching and counters it in Odd-Even (revealed only after you move).
  - **Theory** (`bgPredict`, `bgNormalForm`): P1 wins by matching in both games. pOdd < ½ → two mirror BNEs (1,21) and
    (2,12) (P2 strategy = move in Odd-Even, move in Matching — the slides' order): P1 sticks to a number, Matching type matches, Odd-Even
    type counters; payoffs P1 1−p, P2 1. pOdd > ½ → no pure BNE, P1 mixes 50/50, everyone 0.5. pOdd = ½ → knife-edge.
  - `analyzeBayes`: tiles (matches, realised Odd-Even %, **P1 predictability** = how often Player 1 showed their more
    frequent number, points per round) — below ½ the benchmarks show **pure BNE 100% / P1 0.75 · P2 1** AND the
    **mixed BNE** (everyone 50/50, 0.5 each), which the pure ones Pareto-dominate (captions say so); above ½ only the mixed
    one exists. G11 text uses neutral pronouns ("they"), never he/she for the players. The predictability benchmark for the mixed BNE is
    **what a coin-flipper scores** (`bgCoinPred(L)` = E[max share] over L rounds ≈ 59% at 20, ≈ 62% at 10), not 50%.
    **"Did being predictable pay Player 1?"** groups P1s (80%+ / 61–79% / ≤60%) by P1 points per round (= how often the
    pair showed the same number) vs the BNE. Under the theory 2×4 table, **"Where the class landed"** places each pair in
    the cell of its habit (P1's more frequent number × P2's more frequent move when told Odd-Even and when told Matching),
    with ★ cells and the share on a pure BNE; ties are left out and counted.
  - **CSV**: a bayes row's response is `P1 [moves] · games [M O …] · points k/L · vs Partner` (`bgPoints` from both
    subs + Nature's draws), so an export alone can be analysed. P1 predictability whole/first half/second half vs prediction, P2 "same as P1 in Matching / different
    in Odd-Even" + plan classification (Equilibrium plan / Same number always / Mixed), per-game 2×2 cell shares, the
    Bayesian 2×4 normal form at the round's pOdd with ★ BNEs, lesson note by regime, pairings, orphanWarning.
  - Live-update rule: `studentSubsSig` hides a partner's move for a round I haven't played yet (bayes only), so a
    partner moving first never redraws my screen; verified 10/10 taps under ~20 writes/s.

- `placement` — **Placement Season** (**G12**) — **signalling** (Spence). Candidates are dealt a private type
  (**H** / **L**, high/low productivity) and choose **Grad** or **MBA**; recruiter **firms** (teams of students, one
  **captain** taps) see only the degree and choose an offer **per degree** (Player 2's S₂ᴹᴮᴬ / S₂ᴳʳᵃᵈ). Self-contained
  `ps*` block. Config `{ mode:"fixed"|"bid", variant:"A"|"B"|"C"|"D", firms:5, firmMax:3, recMin:3 }` (`psCfg()`; max 3 recruiters per firm by default since 13 Nov).
  - **Wording:** the job a firm offers a degree is a **role** (Manager / Analyst) everywhere in the UI ("track" survives only
    in code: `PS_TRK`, `res.track`); step 1 is **"Choose role"** (Recruiter or Candidate; "Choose your role for today"), button "1 · Choose role".
  - **Variants** (`PS_VARIANTS`; productivity `prod[type][deg][track]`, recruiter payoff = productivity − salary):
    in **class order** (re-lettered 12 Nov; the sandbox's older records used A/B/D/E and were deleted):
    **A** Classic MBA (50% H, MBA cost H 40 / L 160; Manager 200 / Analyst 100; productivity H 300/150, L 100/100 —
    the MBA adds no skill → separating), **B** MBA builds skills (Manager 300; MBA-trained H 400 / L 350 as managers →
    unique pooling on MBA, belief-free: Manager is dominant after an MBA, Analyst after a Grad), **C** Strong batch (80% H →
    separating + pooling on Grad that holds for any off-path μ), **D** Online MBA (costs 10/40 → pooling on Grad with
    off-path μ ≤ ⅔, μ = 0 cleanest, + the hybrid σ = ½, r = 0.4). Each has a `lesson` line. **B is fixed-mode only** (300 is
    outside the pay band; `psCfg` falls back to A in bid mode and the chip is disabled). A **"This round" banner** (`psVariantBanner`) on candidate (choose + locked), recruiter (choose + recruit)
    and Present screens spells the variant out (MBA costs; "the MBA builds real skills: as a Manager an MBA hire produces H 400 ·
    L 350 (without: 300 · 100)" vs "adds no skill"; Manager pay in fixed mode; % H when ≠ 50) — gold and tagged **NEW THIS
    ROUND**, with the old numbers "(was …)", when it differs from the previous round of the meeting (`psPrevVariant`, from that
    round's subs). Candidates also get the "what a hire produces" table (`psProdTable`).
  - **Modes.** **Fixed**: salary is set by the track; CVs are **dealt** to firms in batches that match the class by
    degree **and** type (deal MBA-H, MBA-L, Grad-H, Grad-L round-robin on one counter, start rotated by round), so
    firms compete on judgement only. **Bid**: offer = track + salary inside **pay bands** (`PS_BAND`: Manager
    200–250, Analyst 100–150 — the bands make every in-band combination separating at costs 40/160); one shared
    pool, **no hiring limit**: every candidate goes to the highest salary for their own degree; a tie at the top splits the
    pool **type by type** (H's in turn, then L's, one counter, seeded start) so each tied firm's H and L counts differ by at
    most one. Bands don't overlap, so a Manager offer always out-pays an Analyst one.
  - **Profit-only DUMMIES** (`psRound` → `firm.dummies`, `realProfit`, `dumProfit`; `PS_DUMMY_NOTE`): an odd split made
    identical strategies score differently (10 Nov sandbox: 100 vs 350 over three bidding rounds with identical offers).
    Every firm that shares a degree's candidates (fixed: all firms; bid: the firms tied at the top) is topped up to the same
    number of H and of L — ceil(n/k) per type — with dummies that produce and are paid like a real hire of that type in that
    firm's role. Dummies count toward **firm profit / leaderboard / profit per hire only** — never the class table, μ/β, the
    Hires count, MBA hit rate, any candidate's result or the CSV. Shown greyed as "Dummy (profit only)" rows in the hires
    table, "+1 dummy" in the KPI panel and leaderboard, with the note. Tied firms share a rank ("Rank 1 of 2 (tied)").
  - **Settings are fixed per round** (`control.ps = {rk, step, mode, variant}`): `openMeeting` stamps the Parameters' mode +
    variant into `control.ps` when a round opens (`psSetStep` keeps the stamp), and `psCfg()` returns the stamped settings for
    the round in play — `psCfgRaw()` is what Parameters is set to. So changing mode/variant after Reveal (or between Close and
    Reopen) only applies from the **next** round; the Parameters card says "Round N runs … — fixed for this round. Next round
    will use …". Why: 13 Nov, Section B — the instructor switched bidding → fixed after round 2; three captains then saved
    round-2 offers in fixed format (no salary), the bidding outcome got NaN salaries, an empty top-offer list, and crashed;
    every round-3 screen that shows last round's report froze on "Dealing your type…". Hardening: `psSal` (bid) treats a
    missing salary as the band floor; `psRound` takes the round's mode/variant from the majority of its records and skips a
    degree nobody offered for; `psLastReport`/`psDebriefCard`/`psRoundsCard`/`psCandidateTable` run through `psSafe`; and
    `render()` catches a crash while building any screen and shows "Something went wrong on this screen — Reload" instead of
    leaving the old screen frozen.
  - **Changing mode / variant** (Parameters): free while Run's round has nothing recorded (also updates the round's
    stamp); **greyed** while a round with data is unrevealed ("reveal it (or discard it) first"); after Reveal a tap asks
    "Switch to … and start Round N+1 now?" and opens it with the new config in the same write (`openMeeting({round, config})`
    stamps `p.config`), with the Run Undo. The leaderboard restarts per mode, as before.
  - **Steps only move forward with a tap** (`psConsoleCard`): the current step isn't clickable, later steps advance
    (`psAdvance`), a finished (✓) step asks "Go back to step …?" first.
  - **Recruiter timer** (`recMin`, default **3** min, 0 = none; Parameters "Recruiter time"): entering step 3 stamps
    `control.ps.recEnd` (server ms; `psSetStep` keeps it on re-entry). Countdown (`psCountdown`, `[data-pscd]`, updated in
    place by the 1 s `rbTick`, which also redraws once when time runs out — `_psTU`, `psTimeUp()` is in `viewKey`) on the
    captain's and teammates' phones, candidates' "choice locked" screen, the console card and the projector. At zero the
    captains' controls lock ("⏱ Time's up — your offers stand"; `psSetOffer`/`psLockOffer` refuse) and the current offers
    are used; the instructor still Reveals. **+1 min** (console card + projector controls, `psAddTime`) extends — or
    restarts from now if it ran out; with no timer it reads "Start a 1-min timer".
  - **"Last round"** (`psLastReport`, `psPrevVariant`) uses `psPrevRk` — the latest earlier round with records that isn't
    discarded (round numbers can skip after a Discard).
  - **Flow** (`control.ps`, `psStep()`): Open → **roles** (first round of a meeting only, when it has
    no firms: every student first sees a **role picker** — 🏢 **Recruiter** (seats left shown; greyed when full) or 🎓
    **Candidate** (writes a ready mark `live/<base>/pc/<enc(id)>` = `{id,name,ts}`, `psPickCandidate`; "Change to recruiter" /
    "I'll be a candidate instead" switch back; **not picking = candidate**). Console and **Present** show live counts
    (`psRoleCounts`: candidates ready · recruiters in N firms · haven't picked yet, the last from the section's enrolled roster).
    Recruiters: volunteers tap **Start a firm** and **name it** — they become its captain, up to `firms` firms, names unique
    and ≤24 chars (`psCreateFirm`); others tap *Join* on a named firm (`psJoin`, cap `firmMax`); captains can rename until
    step 2 (`psRenameFirm`); typing uses `psNewName`/`psRenameTxt` + `rbBlurRedraw`, and firm-list redraws wait while a
    finger is down (`psFirmsRender` checks `_ptrDown`, hook `psPtrHook`) so a classmate starting a firm can't swallow a tap) → **choose** (candidates dealt a type,
    pick Grad/MBA, then **✓ Done — lock my choice** (`lock:true` on the sub; **Edit my choice** reopens it while step 2 runs;
    a choice not marked Done still counts) → **recruit** (choices lock; captains set offers, then **✓ Done — submit our
    offers** (`psLockOffer`, `lock:true` on the captain's sub; controls disabled; **Edit offers** until Reveal; teammates see
    "Locked in by your captain"); bid mode shows a live job board) → Reveal → Next round (goes straight to choose; firms are kept). Step buttons are on the
    console card (`psConsoleCard`: live H/L × MBA/Grad counts; per firm **Rename** / **Remove firm**
    (`psRemoveFirm`), ✕ to remove a recruiter (`psRemoveMember`; last member closes the firm), tap a name to make captain;
    **Reset recruiters** (`psResetFirms`, back to step 1 if open) to swap recruiters between phases or sessions; and, when
    the meeting has no firms, **Copy last meeting's firms** (`psPrevFirms`/`psCopyPrevFirms`, rebuilt from the latest
    earlier meeting's recruiter subs) to keep the same recruiters) and on the Present view; Reveal is locked until the recruit step. `openMeeting` sets `ps`.
    **Who's done** (`psDoneStatus`): the console card lists candidates done / chose but not Done (names) / on the page but
    haven't chosen (names — dealt a type (`live/<rk>/pt/w/<enc(id)>` = `{id,name}`, written when dealt; instructor-only
    listener `psDealtEnsure`) or picked Candidate at step 1, and not in a firm), and in step 3 firms done (+ ✓ done per
    firm card); Present shows the counts (`psLive("canddone"|"firmsdone")`, patched in place). Advancing is never blocked:
    `psAdvance("recruit")` and `psReveal()` (console step buttons, round-bar Reveal, Present buttons) **confirm** when someone
    isn't done ("their choice still counts").
    **Stage tracker** (`psStepper`/`psStageBox`, `PS_STAGES`, `psStageIdx`): four boxes **1 Choose role → 2 Candidates choose →
    3 Recruiters decide → 4 Results** (4 = phase revealed) — done = green ✓, now = gold ▶ + NOW, later = grey dashed ○ (symbol +
    colour, both themes); in rounds 2+ box 1 reads "Roles set (kept)". Students see it at the top of the G12 card and above
    their results, with a tailored **"Now: … · Next: …"** line (`psNowNext`: candidate / recruiter / captain, locked or not).
    The **projector** shows it big across the top — finished stages stay visible as ✓ boxes and are **not tappable** (going
    back is console-only, via the step buttons, which carry the same ✓ / "· NOW" marks). It only changes with the step or
    phase, which already redraw every screen — no new live-update source.
    **Dealing can't hang:** `psEnsureType` gives up after 6 s (`_psTypeErr`) → "Couldn't deal your type" + **Try again**
    (`psRetryType`); a late answer still lands.
  - **Data.** Firms are **meeting-level** (they persist across rounds): `live/<S1A-placement>/pf/<fid>` =
    `{name, captain, members, ts}` (fid generated; `psFirmName(fid, R)` prefers the name recorded in that round's subs,
    so history keeps the name used then), pointer `pr/<enc(id)>` (claimed by transaction, size cap by transaction). Types per round:
    `live/<roundKey>/pt/n` counter → `psTypeFor` (a seeded block of 10 with exactly q% H) → `pt/t/<enc(id)>` (+ `pt/w` names).
    Subs: candidate `{pt:"C", type, deg, mode, variant, lock}`; recruiter `{pt:"F", fid, cap}` (written automatically so
    they count as present), the captain's carries `offer:{MBA:{t,w},Grad:{t,w}}` + `members` (+ `lock` once Done). `lock` is
    a separate field — never `done` (that is `isDone`'s flag). One sub per student
    per round, so being a recruiter and a candidate in the same round is impossible.
  - **Outcome** `psRound(list)` is recomputed from subs only (cached by a signature), so live preview, reveal,
    history and CSV agree; `psRoundsUpTo`/`psLeaderboard` give **cumulative profit per mode** within the meeting
    (fixed and bidding phases have separate leaderboards). Candidates in the choose step and recruiters see **last
    round's placement report** (`psLastReport`).
  - **Reveal** (`analyzePlacement`): candidate's own result (firm, role, salary − MBA cost, phase total) — or, never a silent
    gap, a `psNoResultCard`: "You didn't take part in this round" (no record), "Your firm made no offer this round" (a
    recruiter whose captain set nothing), "Nobody was hired" (no firm made an offer); a
    recruiter's **KPI panel** (cumulative profit + rank, hires by degree, MBA hit rate, profit per hire, avg paid vs
    produced) and hires with types; the class table H/L × MBA/Grad with **μ** (share H among MBAs) and **β**; fixed
    mode states the Manager threshold per degree (`psThreshold`, 2/3 for A) and whether Manager was the best reply;
    bid mode compares salary with what each pool produced; the variant's lesson; the leaderboard. CSV/`subValue`:
    `psSubValue`.
  - **Round debrief — "what you made vs what you could have made"** (plain language; the class meets the game BEFORE
    separating/pooling, μ or β are taught, so separation should show up in the data by itself; averages were dropped as
    misleading). Counterfactuals use the round's ACTUAL offers: a candidate's payoff with the other degree (`psAltC`: bid =
    the top offer for that degree; fixed = the average over firms, since CVs are dealt) vs what they made (`psMadeC`); a firm's
    real profit vs the best it could have made **with the same hires** (`psFirmAlt`: each degree's hires in the better role at
    the band floor (bid) or that role's salary (fixed), or not hiring them = 0; bid notes "assumes you'd still have won them").
    `psDebriefCard(R)` tops every results view. **Instructor console + projector:** named cases — candidate groups (type ×
    degree × payoff) who "left N on the table" (red) and who made the "Best choice" (green), up to 6 names + "and N more";
    every firm's made vs could-have-made (firm names) with the best move — a firm that **hired nobody** is gold "safe this round, not a winning plan" with `psNoHireNote` (which degree's hires earned their firms money at the winning offers, or that none did), never "best possible"; a **Hide names / Show names** link
    (`psDebriefNames`, local); and a ⚠ "That Grad pay came from offers that lost firms N per hire — don't count on it next
    round" on any case whose payoff came from a loss-making offer (so "H as a Grad made 250" doesn't pull H's out of the MBA).
    **A student's phone:** only their own line ("You (H) went MBA and made 210. As a Grad you would have made 250 …") — when the better payoff came from a loss-making offer (`psLossy`) it reads "— but only because firms paid Grads more than they produced (they lost 94 per hire). Don't count on that next round." instead of "40 more", and a best choice paid by such an offer gets the same ⚠
    or their firm's, plus their side's question — never other students' names. Questions (`psPointers`): recruiters — a degree
    paid ≥20 more than it produced → "Who is likely to turn up with a Grad degree?", bid with salary ≈ output → "the bidding
    pushed salaries up…"; candidates — **forward-looking** first: "Firms lost 94 on every Grad hire they paid 250. If they stop
    doing that next round, which choice will pay best for your type?", else H no better with an MBA / L gained from it / "Would
    anyone do better by switching?". Next round's **"Last round's placement report"** (`psLastReport(role)`) shows the
    student's own made-vs-could line (type re-dealt) or the firm's, plus that side's question. Students/projector see "Share
    who are H" instead of μ/β; the variant's theory `lesson` is console-only. **Round by round** (`psRoundsCard` over
    `psMeetingRounds`, from round 2, no names): H/L who took the MBA, avg payoff H/L, MBA/Grad produced/paid, firms' total
    profit + a trend chart of H vs L taking the MBA. **Console only:** `psCandidateTable(R)` — every candidate's type, degree,
    firm, role, salary, MBA cost, payoff and running total, sorted by payoff.
  - **Live-update rule:** `psSubsSig` = my own record incl. its `lock` (+ the captain's offer and lock for a teammate; the
    captain draws from the local `_psOffer` draft); the firms listener has `psViewSig` (`_drawn.ps`). Live counts, done
    counts and the job board are **patched in place** (`psPatchLive`, `[data-ps-live]`) on every subs change — never a full
    redraw. Verified with ~20 writes/s: every candidate toggle, Done/Edit, captain stepper and Done/Edit tap registered and
    the rival board updated live.

## Videos (V1…) — class clips with burned-in subtitles
- **Why not PowerPoint/YouTube:** embedded subtitles didn't reliably show. Each clip is re-encoded once with ffmpeg to
  **H.264/AAC MP4, 720p, `+faststart`, subtitles BURNED INTO the picture** (`-vf subtitles=x.srt`) — so they show on the
  projector, in fullscreen and on iPhones, with no caption track to fail. Example:
  `ffmpeg -i in.mkv -vf "scale=-2:720,subtitles=in.srt:force_style='FontSize=24,Outline=2'" -c:v libx264 -crf 23 -preset slow -c:a aac -b:a 128k -movflags +faststart out.mp4`
- **Hosting:** NOT in this public repo/Pages site (the ferry clip is *The Dark Knight* — copyrighted; a takedown could
  hit the whole portal). **Google Drive** (chosen; free): upload the subtitled MP4, Share → "Anyone with the link", paste
  the link in **Course config → Videos** (`videosCard`). `driveId(url)` pulls the file id from `/file/d/<ID>/…` or
  `?id=<ID>` on a google.com host (folder links are refused), and the player embeds **Drive's own player**
  (`<iframe src=…/file/d/<ID>/preview>`) — Drive blocks using the raw file in a `<video>`. Our Play/Pause/Start over can't
  reach into that cross-origin player, so a Drive video shows only **Full screen** (`requestFullscreen` on the iframe) +
  Exit, and a note to leave Drive's own (auto-generated) CC off. Any other https link (e.g. a Firebase Storage download
  URL — needs the Blaze plan) plays in the portal's own `<video>` player. Stored per course in
  `control.videos = {<vid>:{title,url,ts}}` — the link isn't in the public source; no rules change.
- **The ferry clip** (4:33): subtitles were made with Whisper large-v3 (word timings) and corrected by hand against the
  film; the picture (1280×534) is padded to 1280×720 and the subtitles sit in the bottom black bar (never over the
  picture), the Joker's loudspeaker lines in italics.
- **Playing:** the console's Choose exercise lists them under a **Videos** divider (tiles **V1…**); each is a lesson id
  `"video:<vid>"` (`lessonMode`, `isVideoMode()`), so the Student page tick (`hub.lessonKeys`, list from `allLessons()` =
  `HUB_LESSONS` + videos) and `lessonVisible` gate work unchanged; students open ticked videos from My page's "Lessons &
  videos" card. `videoLesson(vid)`: big `<video controls playsinline preload=auto>`, a buffered-% line ("✓ Fully loaded —
  plays without pauses"), Play/Pause · Full screen (`videoGoFull`, incl. iOS `webkitEnterFullscreen`) · Start over · Exit;
  keys Space / F.
- **Direct link for slides:** `…/?class=<room>&video=<vid>` (`VIDEO_PARAM`, `applyVideoParam` from `onCtrl` once control has
  loaded) opens that video's full-screen page straight away — instructor always; a student only if it's ticked on My page
  (render()'s `lessonVisible` gate). The `video` param is then stripped from the address bar (`history.replaceState`), so
  Exit + refresh lands on the console. An unknown id toasts "That video isn't set up in this course." **🔗 Copy link for
  slides** (`copyVideoLink`/`videoLink`) sits on each Course config → Videos row and on the player (instructor only).
  Full screen still needs one click on the page (browsers block auto-fullscreen).
- **Never reset by a redraw:** ONE `<video>` (or Drive `<iframe>`) element per URL is kept in `_vid` and re-used by every render (moving it
  within one synchronous `replaceChildren` doesn't pause it), and shared-data redraws are skipped while a video is open
  (`onCtrl` returns for `isVideoMode()`, plus the existing lessonMode guards in `teacherRenderSoon`/`scheduleSubsRender`).
  Verified: playback continues through 40 shared writes and a forced `render()`, instructor and student.

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
- **Game trees → normal form** (**IL2**; `treesLesson`, `lessonMode="trees"`, local `TZ` state, `treeInit(key)`) — the
  extensive-to-normal-form conversion in four tabs: **① Information sets** (tap a node → its set; Reveal draws each set
  as a labelled band, e.g. P2·1), **② Subgames** (tap a node to test it — shaded if a subgame starts there, else the
  reason; Reveal lists SG1…), **③ Strategies** (product of actions over the player's information sets, every strategy
  listed, incl. plans for unreached nodes), **④ Normal form** (tap a cell → its payoff + the lit path in the tree;
  Nature's branches give an EXPECTED payoff with the working shown; Fill all; ★ (Bayesian) Nash equilibria). Each tab
  has an "Ask the class" prompt and a Reveal/Hide button. Trees are tiny specs in `TREE_PRESETS` (decision node
  `{p, i, a:[[act, child]]}`, Nature `{p:0, a:[[label, child, prob]]}`, leaf `[u1,u2]`; same player + same `i` = one
  information set): P2 sees P1 / can't see / sees only some moves / plans for unreached nodes / **Bayesian Odd-Even or
  Matching** (G11's games at 25%, same tree as the slide: Nature → P1 → P2) and **Bayesian Tinder BoS** (Tinder reproduces the slides' 2×4 table and BNEs; both currently `hidden:true` — not
  offered in the Examples row until Bayesian games are taught; remove the flag to show them). Engine: `treeBuild`
  (layout, information sets, subgame test), `trStrats`, `trEval`, `treeSVG`. Also on the hub (`HUB_LESSONS` "trees").
  **Lessons are local:** while `lessonMode` is set, shared-data activity (subs, the instructor's control echo) does not
  redraw the screen (`scheduleSubsRender`, `teacherRenderSoon`, `onCtrl` return early), so taps aren't lost.

## Roles & flow
- Student: open link → sign in with Google (or enter a name) → submit → sees own submission +
  a count. Aggregates are hidden until the instructor reveals.
- Instructor: click "Instructor" → passcode → console (pick game, open/close submissions,
  reveal, next round, clear round data, edit parameters). First run sets the passcode.
- **Console layout** (`teacherConsole`) — a **single full-width column** (`.console`,
  `grid-template-areas:"sw" "top" "main"`, **no side rail**): the **Run / Review switch** (`.console-sw`), then in Run a
  **full-width Choose exercise** picker (`.console-top` — the game tiles + the IL1/IL2 lessons + videos) over the
  **main** panel: the chosen game's **Parameters** card, the unscheduled-day/schedule-Undo banner, the session label +
  section/hold bar (`sessionTile`, no session stepper), the "↶ Back to it" Run-undo banner, the 🔒 line when open, the
  round bar (Round label · Open/Close · Reveal · Next round → · Present · Discard…), the today-only **Analyze** bar,
  then live preview/analysis and the attendance card. Review replaces all of that with `reviewConsole`.
- **Top bar** (`.topbar`, a flex **column** of `.tb-row`s) — **Row 1:** the brand (left) +
  (`.tb-actions`, right-aligned) the phase pill, **Log out** (`#logoutBtn`/`syncLogoutBtn`,
  instructor-only → `logOut`) and the theme toggle. **Row 2** (`.tb-sub`): the **course picker**
  `<select>` (left) + (`.tb-actions`, right) **📋 Take roll call** (`#rollBtn`/`syncRollBtn`),
  **⚙ Course config** (`#cfgBtn`/`syncCfgBtn`) — both instructor-only, hidden in any full-screen
  mode — and the role toggle (`#roleBtn`, Instructor / Exit console). Each row's actions wrap and
  stay right-aligned, so nothing overflows.
- **Course config page** (`courseConfig`, local `cfgMode`) — a full-screen setup page (opened by the
  top-bar ⚙) holding the one-time cards moved off the rail, **stacked full-width** (`.cfg-grid`,
  flex column): **Course** (`courseCard`), **Course roster** (`rosterEditor`), **Student page**
  (`studentPageCard` — the `control.hub` config), **View as student (read-only)** (launches the
  diagnostic below), **Session schedule** (`scheduleEditor`) and
  **Data** (CSV export). The old **Share with students** and **Testing sandbox** rail cards were
  removed (the sandbox is still reachable via `?class=sandbox`). Keeps the console focused on
  running class.
- **View as student (read-only)** (`diagView`, global `diagAsId`; `diagOn()`) — an instructor
  diagnostic that renders the whole app **as a chosen student** so you can reproduce what they
  report ("I can't see my result", "my submission is missing"). Launched from the Course config
  card; a picker (roster by **Name · PGP id (Section)**) sits above the student's own view. It
  works by impersonating identity in `applyIdentity` (when `diagAsId` is set, `myId/myName/meGuest`
  become the chosen student's) and making `isTeacher()` return `false` so every student branch
  renders as they'd see it. **Read-only is enforced at the data layer, not per-button:** `boot()`
  wraps the DB in `guardDb()`, whose `set/update/remove`/`onDisconnect` become no-ops whenever
  `diagOn()` — a single choke point, so nothing in the impersonated view can ever write. Exit clears
  `diagAsId`. `diagAsId` is part of `viewKey()` so switching students re-renders.
- **Collection view** (`presentView`, local `presentMode` flag) — a full-screen, class-facing
  screen that auto-opens when the instructor presses **Open submissions** (also via the round
  bar's **Present** button). While submissions are open/closed it shows participation only — live count,
  progress vs the section's roster, and a **join QR** (`qrSVG`/`qrFor` of `location.href`) — never a result.
  **Reveal keeps it open and switches it to Results** (`presentResults`; `doReveal` no longer clears
  `presentMode` — a Reveal pressed on the console with the present view closed still stays on the console):
  the class-level analysis over the console's meeting scope (`analysisSubs()`/`effScope()`), drawn with the
  student-safe `analysisBody(g, list, false)` and a global **`_presenting`** flag so nothing name-by-name
  reaches the projector — `pairingsCard` returns an empty div, `orphanWarning` returns null, G12's personal
  "Your result"/`psNoResultCard` is skipped, and team/firm member lists are already teacher-only. The Guess
  winner's name still shows (intended). Full instructor detail stays on the console. The scroll position is
  kept across the ~1/s console redraws (`_presScroll`, keyed by roundKey + scope), and `.present-controls` is
  **sticky** at the bottom (Hide results · Next round → · Fullscreen · Exit to console), so long results never
  push the buttons off screen. G12 keeps its stage tracker on top ("4 Results").

## Conventions
- **Live-update rendering rule (MANDATORY for every game, existing and new).** `render()` rebuilds the whole
  screen, so a redraw replaces every button. Other people's activity — a classmate's move, another team's edit,
  a roll-call tick, a heartbeat — must **never redraw a student's screen unless something THAT student sees
  changed**. With a full class these events arrive several times a second; redrawing on each one replaced the
  button under the student's finger and taps were lost (G8 "can't click after round 10", the waiting-room
  blink). How it's enforced (keep using these, don't call `render()` straight from a shared listener):
  - Each shared listener checks a **view signature** of what this viewer sees and returns early if it equals
    what's on screen: `studentSubsSig()` (subs, in a live pair game: own sub, partner's sub, class done-count,
    pair id), `rbViewSig()` (G10 teams: the team list while choosing, then only your own team; the captain's
    own robot echo excluded), `studentCtrlSig()` (control minus roll-call `attend` unless My page shows it),
    `lobbyViewSig()` (waiting room: present players + incoming invite, not heartbeat `ts`). `render()` records
    the drawn signatures in `_drawn` (and `pairLobby` records `_lobbySig`).
  - The instructor console is throttled to ~1 shared-data redraw per second (`teacherRenderSoon`).
  - Local actions (your own tap) may call `render()` directly; echoes of your own write must not cause a second
    redraw if the screen already shows them (draw from the local draft and leave that field out of the sig).
  - **A new game that shows other people's live data must add exactly those fields to the relevant signature**,
    and be checked under load: simulate ~20 shared writes/s and confirm human-speed taps (mouse down → 120 ms →
    up) all register and that genuinely relevant changes (partner's move, your team) still appear instantly.
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
3. **Bump `APP_VERSION`** (near the top of `index.html`) so returning students get the "new version
   — Refresh" banner, then `git add index.html && git commit -m "..." && git push`.
4. Wait ~1 min for Pages, then hard-refresh (Ctrl+F5) — Pages/browser can cache the old file.

## Likely next tasks
- Add a game (sealed-bid auction, Cournot/Bertrand): extend `GAMES`, `DEFAULT_CONFIG`, add a
  `student<Game>()` view, an `analyze<Game>()` function, and a branch in `paramEditor()`.
  Follow the **Live-update rendering rule** (view signatures + load test) for anything that listens to
  other people's data; prefer self-contained function blocks (like `adMatch`/`rpMatch`/`rb*`).
- Optional hardening: tighten Firebase rules or add Firebase App Check to limit who can write.
- A per-round CSV export, or a persistent leaderboard across rounds.

## Licence
`LICENSE` — **all rights reserved** (© 2026 Prof. Sonia, IIM Lucknow — sole copyright holder). Enrolled students may use
the hosted app; **educational reuse only with explicit written permission** (request via a GitHub issue / @eclairs20);
everything else needs permission. The footer carries a matching © line linking to `LICENSE`. The inlined
qrcode-generator stays MIT. Keep the © line when editing the footer.

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
