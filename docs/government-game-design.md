# Form the Government — design for next year (not built)

Status: **the full version below is designed, not built.** On 25 Nov 2026 a simplified version was built and played as
**G14** instead: groups of 5 students, one party each (fixed fictional parties A–E), 100 seats with 51 to govern, only three
posts (PM 50%, Home 30%, Finance 20% of the power — so it stays a simple game), round 1 = 27/26/25/12/10, round 2 =
40/20/15/15/10, a 6-minute class clock with up to two automatic +1-minute extensions (see CLAUDE.md, `gov`). The full version
here remains the plan for a later batch.
Everything here was discussed with the instructor; items marked **OPEN** still need a decision.

## Teaching goal and place in the course

- Teaches **power in voting games**: seats ≠ power, Shapley–Shubik vs Banzhaf, dummy players, and that a simple game
  with no veto player has an **empty core** (deck 14, slide 20: "a simple game has a non-empty core iff it has a veto
  player"). Deals keep breaking until the clock forces one, so the class experiences the empty core.
- **Play it BEFORE teaching the Shapley value.** The class learns the core (with G13's results), plays this game at the
  end of that session, and the next session opens with its results. Seeing their own data motivates "what does each
  party deserve?" → Shapley value → applied to voting = Shapley–Shubik → Nassau County → Banzhaf. Playing it after
  Shapley would test recall rather than instinct.
- **Students never see power numbers during play**, only seats, the majority line and the portfolios. All theory is shown
  at reveal.

## Setting (Indian context)

- A **Lok Sabha of 543 seats, 272 to govern**.
- **Party names (decided: fictional but recognisably Indian)**, e.g. Rashtriya Vikas Party, Lok Shakti Dal, Jan Morcha,
  Kisan Mazdoor Party, Samata Sangh, Pragati Front, each with a colour and symbol.
  - **Why not real party names in play:** students would be assigned parties they may oppose, every seat map casts a
    real party as "dominant" or "dummy", and the app is public, so screenshots could spread out of context.
  - **Real data goes in the debrief instead:** a 2024 Lok Sabha card (see Results).
- **Indian concepts that map onto the theory:**
  - **Anti-defection law (Tenth Schedule):** a party votes as one bloc, so its leader confirms for the party.
  - **Outside support:** a coalition partner with 0 portfolios (e.g. the Left with UPA-I). **OPEN:** allow it?
    Recommendation: yes, labelled "outside support".
  - **Kingmakers**, the **Speaker**, and a **hung parliament** if no government forms in time.

## Seat maps: three parallel parliaments ("houses") per section

With 6 parties in one house the "power ≠ seats" punchline shows clearly. With 10+ parties power ≈ seat share and the
lesson disappears (checked). So for ~60 students, use **3 houses × 6 parties × 3–4 students** (2 houses for ~45),
each house with a different seat map. Same maps in every section, so each map gets one observation per section.

| House | Seats (sum 543) | Shapley–Shubik % | Banzhaf % (normalised) | What it shows |
|---|---|---|---|---|
| 1 | 179 / 147 / 109 / 54 / 33 / 21 | 37 / 27 / 27 / 3 / 3 / 3 | 35 / 27 / 27 / 4 / 4 / 4 | Unequal seats, equal power (109 = 147; 54 = 21) |
| 2 | 217 / 82 / 71 / 65 / 59 / 49 | 60 / 10 / 10 / 10 / 10 / **0** | 64 / 9 / 9 / 9 / 9 / **0** | A dominant party and a **dummy** |
| 3 | 109 / 103 / 98 / 92 / 87 / 54 | 20 each / **0** | 20 each / **0** | Five equal powers and a 54-seat **dummy** |

(The same structure at 100 seats / 51 to govern: 33/27/20/10/6/4, 40/15/13/12/11/9, 20/19/18/17/16/10. Presets should
also include the deck's 45/25/15/15 at 51 and Nassau County 31/31/28/21/2/2 at 58.)

Because a single government is one observation, the class **cannot** tell Shapley–Shubik from Banzhaf (they differ by
≤ 4 points on these maps). The testable question is **"did points follow power or seats?"**

## The prize: cabinet portfolios (symbolic points, editable)

| Portfolio | Points |
|---|---|
| Prime Minister | 25 |
| Home | 10 |
| Finance | 10 |
| Defence | 9 |
| External Affairs | 8 |
| Railways | 6 |
| Road Transport & Highways | 5 |
| Agriculture | 5 |
| Commerce & Industry | 5 |
| Health | 4 |
| Education | 4 |
| Rural Development | 4 |
| Speaker of the Lok Sabha | 5 |
| **Total** | **100** |

The PM post is indivisible (25 points to one party), which is a talking point in itself. Weighted portfolios are how
Warwick & Druckman tested Gamson's law.

## Round flow (one round, ~12–14 min, at the end of a session)

1. **Open → "I'm here"** (~1 min). The app **auto-deals** present students at random into houses and parties, so nobody
   picks the big party. The first to tap **"I'll lead"** in each party is its leader; otherwise one is chosen at random.
   Late arrivals join the smallest party in the smallest house; the instructor can move people.
2. **Read your party** (~1 min): house, party name, seats, the 272 line, portfolios. No power numbers.
3. **Negotiate** (8–10 min, timer + "+1 min").
   - **Only leaders propose:** a proposal = which parties are in + which party gets each portfolio (tap a portfolio,
     then the party).
   - **A leader backs one proposal at a time.** Confirming another withdraws the first, which is how deals break.
     The proposer's party confirms automatically.
   - **Every other member** sees their house's proposals live and taps 👍/👎 on any that include their party
     (advisory; the leader sees "4 of 5 back this").
   - **"Are you sure?"** (as in G13) when a proposal is short of 272, or carries a party it doesn't need (not
     minimal winning, e.g. including a dummy).
4. **Government forms** when a proposal with **≥ 272 seats** is confirmed by every party in it. It locks; no toppling.
   Parties outside get 0. **Timer ends with no government → hung parliament, everyone 0.**
5. **Projector:** per house, a seat bar for each live proposal racing to 272, ✓ marks as leaders confirm, a count of
   broken deals. Parties and seats only.

Decided: hung parliament = 0; only leaders propose; no toppling; one round; the 👍/👎 party vote; auto-dealt parties.
**OPEN:** outside support; whether team size should be equal (recommended) or in proportion to seats.

## Results (revealed the next session)

- Per house: the government formed, its seats, **minimal winning or not**, the number of broken deals.
- Per party: **seat share vs Shapley–Shubik (in points = 100 × index) vs Banzhaf (normalised) vs points actually won**,
  plus **Gamson's law** (points ∝ seats within the coalition) and **proposer advantage** (did the party that made the
  winning proposal keep more?). Question for the class: "Did you split by seats, by power, or did the proposer grab it?"
- Show Shapley–Shubik first (it is the Shapley value of the voting game, so it follows the teaching order), Banzhaf as a
  second view ("share of decisive votes").
- **Nassau County card:** 31/31/28/21/2/2 at 58 → three dummies with 21% of the votes under either index.
- **2024 Lok Sabha card** (real data, debrief only; **verify the seat numbers before use**). A rough calculation with the
  main parties and the rest as one bloc gave BJP 240 seats (44%) → ~63% SS / ~75% Banzhaf (a real case where the indices
  differ), INC 99 → ~7%, TDP 16 / JD(U) 12 → ~2% each. Lesson: indices assume any coalition can form; ideology and
  pre-poll alliances restrict that, which is what made TDP and JD(U) pivotal in practice.

## Theory notes for the debrief

- Shapley–Shubik = the Shapley value of the simple game (marginal contribution is 1 exactly when pivotal), so "share of
  orderings in which you are pivotal" — **P-power** (share of the prize). Shapley & Shubik, APSR 1954.
- Banzhaf = share of coalitions in which you swing the result, each coalition counted once; raw values don't sum to 1 —
  **I-power** (influence over a yes/no vote). Banzhaf 1965 (*Weighted Voting Doesn't Work*, Nassau County); Penrose 1946.
  For Nassau the two indices agree; Banzhaf's contribution was the legal framing ("one person, one vote").
- Example where they differ: quota 3, weights 2-1-1 → SS ⅔, ⅙, ⅙; Banzhaf ⅗, ⅕, ⅕.
- Gamson's law (1961; Browne & Franklin 1973; Warwick & Druckman): real cabinets split portfolios ∝ seats, against
  bargaining theory's proposer advantage (Baron–Ferejohn) — the "Gamson's law paradox".

## Build notes

- Reuse: G13's propose/confirm and "Are you sure?" (`ng*`), G10/G12 team machinery (transaction-claimed pointers,
  captain/leader, timers with +1 min, `rbTick`), G12's live offer board patched in place (`psPatchLive`).
- Data sketch: meeting-level `live/<S…-gov>/h/<house>/p/<party>` = `{name, seats, members, leader}`; per-round proposals
  `live/<roundKey>/gp/<house>/<pid>` = `{by, parties, alloc:{portfolio:party}, ok:{party:true}, ts}`; a leader's single
  backing as a transaction-claimed pointer; each student writes their own sub (lockdown rules).
- Follow the **live-update rendering rule**: a student's view signature = their house's proposals + their own party, never
  other houses; load-test at ~20 writes/s. Never put a path and its descendant in one multi-path update (Firebase rejects
  it; the test harness now checks this).
- Power indices: compute exactly by dynamic programming over (coalition size, seat total) per player; instant for 6–10
  parties.
- Estimated effort: ~2 days, plus a sandbox run with real phones before class.
