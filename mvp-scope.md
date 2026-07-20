# MVP Scope — Complete a Game + Live Overlay

**Status:** draft for review
**Note:** A design document. It references an internal requirements document that is not part
of this public repository; the game rules are in [game-rules.md](./game-rules.md).

This document defines the first deliverable only. It deliberately does not restate the
state machine or the rules — those live in [game-rules.md](./game-rules.md) and are cited here by ID.

---

## 1. The milestone in one sentence

A judge can create a game, seat ten players, record the roles they drew, run the game
through to a win condition, and a separate read-only overlay renders the live game state —
including roles — for a stream audience.

---

## 2. In scope

- Create a game; seat 10 players (free-text nicknames are sufficient)
- Judge records the physical card draw (6 Civilian / 2 Mafia / 1 Don / 1 Sheriff)
- Full day/night cycle per the `game-rules.md` state machine (S1–S19)
- Speeches with timer; nominations; voting including tie escalation and all-out vote
- Night actions: Mafia shot, Don check, Sheriff check, best move
- Win-condition evaluation and game end
- Every action recorded as a typed, editable record (nominations, votes, eliminations, night actions)
- Corrections are edits: the judge can fix or delete any recorded action, and can directly
  override any seat's state, role, or fouls (D8)
- State-machine transitions happen only on explicit judge confirmation (D10)
- Live overlay: seat numbers, nicknames, alive/dead, day/night, roles (roles view only, D7); no timer
- Two staff roles: judges run their own games, admins have full access to all games (D9)

## 3. Out of scope

Explicitly deferred, not forgotten:

- **Scoring, points, ratings** — blocked on a client scoring document anyway (`requirements.md` §7.5)
- **Clubs, tournaments, multi-table, seat allocation** — single standalone game only
- Player accounts, profiles, photos, public federation website
- Prizes, currency handling, participation fees
- Appeals, disqualification workflow, protocol export

Fouls are recorded as **counters only** (a judge can increment a player's foul count) with
no automatic consequence. The full foul catalogue and its penalties are out.

---

## 4. Decisions locked

| # | Decision | Rationale |
|---|---|---|
| D1 | **Judge records the physical card draw.** App does not deal roles. | Matches rulebook; players draw cards. Avoids a rules dispute on stream. |
| D2 | **Build the red (proposed) ruleset**, stamped per game via `ruleset_version`. | Federation's direction of travel; mostly additive. Version stamping means an old game always replays under the rules it was played by. |
| D3 | **Rule variants are per-game settings, not code branches.** | Judge discretion and ruleset drift are permanent facts of this domain. |
| D4 | **Normalized relational model.** Every game action is a typed row (nominations, vote rounds, eliminations, night actions); a seat's alive/dead and the win condition are recomputed from those rows, never maintained by hand. | The typed tables *are* the game history — scoring reads them directly when it lands later, and Postgres rows give the durability `requirements.md` ranks #1. Replaces an earlier event-sourced design: same information, less machinery, and corrections become ordinary edits (D8). |
| D5 | **Overlay is a separate read-only endpoint**, not a mode of the judge app. | Role leakage to a player destroys the game. This is a security boundary, not a view toggle. |
| D6 | **Votes stored as aggregate counts**, not per-voter. Judge enters a tally per candidate; individual voter identity is not captured. | Per-voter is impractical during a ~1.5s show of hands. Consequence: scoring rules needing vote identity (e.g. 8.5.1.1) cannot be auto-derived — they would require separate manual judge entry if ever adopted. Resolves OQ-D. |
| D7 | **Two overlay views per game** — a *safe* view that never shows roles and a *roles* view that always does, each behind its own unguessable token. | The overlay is a view, so role visibility is a property of the URL loaded, not mutable game state. Replaces the earlier `show_roles_on_overlay` game setting and removes any mid-game toggle ambiguity: the stream operator simply loads the view they want. |
| D8 | **Corrections are edit-in-place.** The judge can edit or delete any recorded action at any time — "pressed 4, meant 3" is fixed by editing that record, and derived state recomputes. The judge can also directly override any seat's state, role, or foul count as the final word. | Judges make live mistakes and fixing them must be simple. An undo stack forces unwinding every intervening action; editing the wrong record directly does not. |
| D9 | **Two staff roles: judge and admin.** A judge runs their own games; an admin has full access to every game, including taking over a live one. | Matches federation staffing, and covers a judge dropping mid-game without needing a separate handoff protocol. |
| D10 | **State-machine transitions require explicit judge confirmation.** The app proposes the next phase; nothing advances until the judge confirms. | A mis-tap must never cascade the game forward on its own; the judge remains the authority the rulebook says they are. |

Trade-off accepted with D4/D8: there is no append-only audit trail of judge edits. If
dispute-proofing is ever needed, a generic DB history trigger adds it without app changes.

---

## 5. Game settings

Per-game, set at creation, immutable once the game starts:

| Setting | Default | Effect |
|---|---|---|
| `ruleset_version` | `emf-2026-07-proposed` | Which rule set this game was played under |
| `allow_split_3_at_9` | `false` | Whether an all-out vote may eliminate three players when nine remain (GR-65). Judge-discretionary in practice; ruleset-dependent. |
| `speech_seconds` | `60` | GR timing table |
| `tie_speech_seconds` | `30` | GR-58 |

Role visibility on the overlay is deliberately **not** a game setting — it is a property of
which overlay view is loaded (D7).

`allow_split_3_at_9` is the first of what will be several rule-variant flags. The settings
model should be extensible without migration — expect more.

---

## 6. The elimination-set problem

Day elimination is **a set, not a single player**. The all-out vote (GR-61–62) can remove
two or more players simultaneously. Any model assuming one-elimination-per-day will need
rework, so it is specified correctly from the start.

Consequence for final words: resolved — see §8 OQ-A.

---

## 7. Overlay security requirements

Overlays are for viewers only and never interfere with the table — phones are prohibited at
the table, so players cannot see them during play. Requirements:

- **O1** — Each overlay view (D7) is a distinct endpoint with its own unguessable per-game
  token. The roles view and the safe view never share a token.
- **O2** — Role data is filtered **server-side**, keyed off which token was presented. Roles
  are never sent to a safe-view client and hidden with CSS or client-side JS.
- **O3** — The safe view is what game creation surfaces by default; the roles URL is a
  separate artifact the judge hands only to the stream operator.
- **O4** — Rule 4.4.16 is a **table rule**: the *players* must not learn whether a dead
  player was Mafia, Sheriff, or Don. It imposes nothing on overlay content. The roles view
  shows every seat's role — including dead seats — for the whole game; the safe view shows
  none, ever.
- **O5** — Overlay is strictly read-only. It can never mutate game state, and overlay
  clients must not be able to publish to the live-update channel (see technical design §4).

The overlay shows no speech timer — timing lives on the judge's screen only.

---

## 8. Open questions for the client

| ID | Question | Blocks |
|---|---|---|
| ~~OQ-A~~ | **RESOLVED: each eliminated player gets their own 60s final word** (a triple elimination = up to 3 minutes of sequential speeches). Order: **nomination order** `[assumed, matches GR-59]` — confirm with client. State machine: `FINAL_WORD_DAY` loops over the eliminated set. | Day-phase flow |
| OQ-B | Red vs black ruleset — is the proposed text in force for the tournaments this will judge? D2 assumes yes. | Confirms D2 |
| OQ-C | Should `allow_split_3_at_9` default on or off? | Default value only |
| ~~OQ-D~~ | **RESOLVED (D6): aggregate counts.** See §4. | — |

OQ-A and OQ-D are both resolved. Remaining open questions (OQ-B red-vs-black ruleset,
OQ-C split-3 default) are non-blocking and can be answered later without rework.

---

## 9. Build order

1. **Data model + derived-state recompute** — the spine. Nothing else works without it.
2. **Game creation + seating + role recording** (D1)
3. **Day phase** — speeches, timer, nominations, voting, tie escalation, all-out vote
4. **Night phase** — shot, Don check, Sheriff check, best move
5. **Win conditions + game end**
6. **Overlay** — read-only view over the same data

Steps 1 and 6 are the two with architectural consequences. 2–5 are mostly a faithful
transcription of the `game-rules.md` state machine.

---

## 10. Stack (locked)

**Next.js (App Router) on Vercel + Supabase Cloud (Postgres, Realtime, RLS) + Supabase
Auth.** BaaS-first for minimal maintenance. TypeScript, not Python. See `CLAUDE.md` for the
architecture notes — chiefly that RLS is the role-security boundary (D5), and the overlay
receives its live updates via Realtime Broadcast.

## 11. Not yet decided

**Rebuild vs fresh product.** Whether this is a reskin of an existing reference product or an
independent build still shapes UI decisions, though less so for the MVP than for later phases.
