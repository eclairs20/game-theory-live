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
  When today's date matches, `scheduleBanner()` (top of the console main pane) offers a **tap-to-apply**
  reminder — one button per scheduled `(session, section)`, or a session-only button when no section is
  listed — that `writeControl`s the session/section. It **never auto-writes** `control`; the instructor
  taps to apply (or `dismiss`). Section-less rows nag only on a session mismatch (the common
  both-sections-same-day case); section rows also nag if the running section isn't scheduled that day.
  Edited via the console's **Session schedule** card (`scheduleEditor`, paste like the roster).
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
  "X of N enrolled present" count via `enrolledRoster`). The header carries the session −/+ stepper
  inline and shows the meeting's **date** from the session schedule (`schedDateFor`/`fmtSchedDate`,
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
    `sid`) × every session `1..N` (`gridSessions(sec)` — covers taken, scheduled and current). Sticky
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
    enter the match. Works for **guests** too — matching is by presence, not roster identity. The
    listeners attach via `ensureLive(mc)` and are torn down by `detachLive()` (which also removes
    your lobby presence) whenever `render()` sees you're no longer on an open pair game.
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
    `isDone(s) && !s.bot`).
  - **Practice bot** — a "🤖 Practice bot" button in the lobby sets `d.bot` and drops you straight
    into a match against a locally-generated ~Nash opponent (`botRBSeq`/`botPiles`), revealed hand
    by hand as you play; on completion `saveBot()` writes a synthetic `"bot:<myId>"` opponent sub so
    the analysis sees a full pair. Bot docs are `bot:true`/`guest:true`, excluded from counts.
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
    open. You can never be more than one unresolved round ahead of your partner. **Restart**
    (`liveLeaveMatch`) in a live pair match clears **both** sides — your sub AND the partner's own sub
    (+ `of` pointer, + the pair's `flips`) — so neither reloads stale moves or lingers as a phantom
    submission, and removing the partner's `of` drops them straight back to the waiting room.
  - Reuses the G5/G6 **lobby/pairing** machinery (`studentPair("attackdefend")` → `pairLobby` →
    `adMatch`), but is a **self-contained** function block (`adMatch`/`adBattle`/`adTroopIcon`/
    `adSoldier`/`adShield`/`adCoin`/`adPlaceDesc`/`analyzeAttackDefend` + the `AD_*`/`ad*` helpers)
    that does **not** touch `pairMatch`, so the card games can't regress. **Live-only** (`wantLive`).
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
    BREACHED"** or flies a **green flag + "🛡 TOWN HELD"**, SMIL `<animate>`); **interactive placement**
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
- **Console layout** (`teacherConsole`) — a **single full-width column** (`.console`,
  `grid-template-areas:"top" "main"`, **no side rail**): a **full-width Choose exercise** picker
  (`.console-top` — the G1–G6 game tiles + the IL1 Pareto lesson) over the **main** panel. The
  chosen game's **Parameters** card stacks at the **top of the main panel** (first card, full width),
  followed by session/round controls, instructor preview/analysis and the attendance card.
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
3. **Bump `APP_VERSION`** (near the top of `index.html`) so returning students get the "new version
   — Refresh" banner, then `git add index.html && git commit -m "..." && git push`.
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
