# Security & Firebase rules

The Realtime Database started fully public (`.read: true, .write: true`) — fine to
launch a classroom, but it means anyone who finds the URL can read or overwrite
anything. This is the plan to lock it down. Rules live in the **Firebase console**
(Realtime Database → Rules) and must be pasted there by the project owner — they
are not deployed from this repo automatically.

> **Rooms:** the app now namespaces data per class via `?class=<id>` (e.g. `pgp-2026`),
> so both rule files apply to **every** room via the `$room` wildcard (not just `main`).
> If you already published the `main`-only rules, re-publish the updated file once so the
> new class rooms are covered.
>
> **Global `courses` / `enroll` nodes:** the course picker and student auto-routing add two
> top-level nodes outside any room — `courses/<id>` (the course registry) and
> `enroll/<email>` (which courses each student is enrolled in). Both rule files now include
> them: open read/write under the stopgap; under lockdown they are readable to any signed-in
> user and writable only by the allowlisted instructor emails (same list as `control`/`roster`).
> **Re-publish whichever tier you're on** after this change, or creating a course / saving a
> roster will be denied.
>
> **Per-room `schedule` node:** the optional session schedule (date → session/section, for the
> console's tap-to-apply reminder) is stored at `‹room›/schedule`. Both rule files now include it,
> mirroring `roster` (open under the stopgap; instructor-only write under lockdown). **Re-publish
> your tier** after pulling this change, or saving a schedule will be denied.
>
> **Per-room `live` node:** the real-time waiting room for the pair card games (G5/G6) stores
> ephemeral coordination at `‹room›/live/‹mc›` — lobby presence, invites, id→pair pointers, and pair
> records. Both rule files include it: open under the stopgap; under lockdown **any signed-in,
> email-verified user** may read/write it (students, including guests, coordinate matches there —
> this is not instructor-only). The actual moves still live in `subs` under the existing rules.
> **Re-publish your tier**, or the waiting room can't pair anyone.

## Two tiers

| File | When to publish | What it does |
|------|-----------------|--------------|
| `firebase-rules.stopgap.json` | **Now** — safe immediately | Blast-radius fix. Still open (no sign-in required), but the database root and unknown paths become unwritable (no full wipes / junk nodes), and `control` + submissions must have the right shape. Names length-capped. |
| `firebase-rules.lockdown.json` | **Later**, once the checklist below is all true | Real write- **and read-security**. Only signed-in Google users can read or write (the app is sign-in-first, so it never reads before authenticating); every write requires a verified Google account; students can only submit as their own verified email; only allowlisted instructor emails can change the game/round/phase/config or the roster. |

## Apply the stopgap now
1. Firebase console → project **game-theory-live** → Realtime Database → **Rules**.
2. Paste the contents of `firebase-rules.stopgap.json` (drop the `//…` comment keys if
   the editor objects to them). Click **Publish**.
3. The app keeps working exactly as today.

## Switch to full lockdown (later)
Do **all** of these, in order, or people get locked out:
1. **Enable Google sign-in** — Authentication → Sign-in method → enable **Google**;
   Authentication → Settings → Authorized domains → add `eclairs20.github.io`.
   Confirm you can actually sign in on the live site.
2. **Load the class roster** in the instructor console.
3. **Every instructor's Google email must be in two places** so they aren't locked out
   (currently `ankit.khandelwal@gmail.com` and `sonia@iiml.ac.in`):
   - `INSTRUCTOR_EMAILS` in `index.html`, and
   - the `control` and `roster` `.write` rules in `firebase-rules.lockdown.json`.
   Add any additional instructor to both spots before publishing.
4. **Set `REQUIRE_GOOGLE = true`** in `index.html`, commit, and push (Pages redeploys).
   This makes student joining Google-only and instructor access allowlist-only
   (no passcode) — matching what the rules enforce.
5. **Publish `firebase-rules.lockdown.json`** in the console (Rules → paste → Publish).

To roll back at any point: re-publish `firebase-rules.stopgap.json` and set
`REQUIRE_GOOGLE = false`.

## What each tier does and doesn't protect
- **Stopgap:** stops catastrophic writes (root wipe, arbitrary new paths) and malformed
  data. It does **not** stop a signed-out person from submitting or editing `control`
  through the normal paths — the instructor passcode is still only a front-end gate.
- **Lockdown:** the game state (`control`) and roster can only be changed by the
  allowlisted instructor accounts, enforced by Google's servers — the real fix.
  Students can only write submissions stamped with their own verified email. Reads
  now require sign-in too (`.read: auth != null`): the app is sign-in-first, so it
  never reads before authenticating, and the anonymous public can no longer read the
  roster or submissions. Residual: any *signed-in* user (including a guest) can still
  read all data — hiding it between students would need Option B (uid-keyed
  submissions + an instructor-only email node).
  Residual: a submission's storage key isn't cryptographically bound to the writer, so
  a determined signed-in student could overwrite another submission key while still
  stamping their own email on the record (so it's traceable). Closing that fully would
  need Cloud Functions or uid-keyed writes — overkill for an in-class exercise.
