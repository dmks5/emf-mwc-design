# Technical Design — MVP (Complete a Game + Overlay)

**Stack:** Next.js (App Router) on Vercel · Supabase Cloud (Postgres, Realtime, RLS) ·
Supabase Auth. See [mvp-scope.md](./mvp-scope.md) for scope and locked decisions D1–D10.

This document covers the data model and the role-security boundary for the MVP only.
Game-flow logic (the 19-state machine) lives in TypeScript in the Next.js app and is
specified in [game-rules.md](./game-rules.md) §1 — not repeated here.

---

## 1. Surfaces and who talks to what

Two clients, one backend:

- **Control panel** — authenticated (Supabase Auth), used by two staff roles (D9): a
  **judge** sees their own games, an **admin** sees every game and can take one over
  mid-game. Talks directly to Postgres via the Supabase client, constrained by RLS. Holds
  the game-flow logic; writes typed action records, and each write runs the derived-state
  recompute (§2). Phase transitions require the judge's explicit confirmation (D10).
- **Stream overlay** — unauthenticated, reached by an unguessable per-game URL token. Two
  views per game (D7): a **safe** view that never shows roles and a **roles** view that
  always does, each behind its own token. **Neither ever reads roles from the database.**
  Each gets its initial state from the Next.js server (service role, roles pre-filtered per
  view) and live updates via a Realtime Broadcast channel whose payload the judge's client
  controls.

The asymmetry is deliberate: the judge is a trusted authenticated actor; the overlay is a
public surface and is treated as hostile by default.

---

## 2. Data model

Normalized (D4): typed tables record every game action and *are* the game history. A seat's
alive/dead state and the win condition are recomputed from those records — never maintained
by hand — though the judge can directly override seat state, role, and fouls (D8). No
sequence column is needed: the rulebook itself fixes the order of phases within a day.

```sql
-- Staff roles (D9). Every authenticated user is a judge or an admin.
create table profiles (
  id    uuid primary key references auth.users(id),
  role  text not null default 'judge' check (role in ('judge','admin'))
);

-- security definer so RLS on profiles doesn't block the check itself.
create function is_admin() returns boolean
  language sql stable security definer set search_path = public as
  $$ select exists (select 1 from profiles where id = auth.uid() and role = 'admin') $$;

-- One row per game, owned by the judge who created it.
create table games (
  id                    uuid primary key default gen_random_uuid(),
  judge_id              uuid not null references auth.users(id),
  status                text not null default 'setup'
                          check (status in ('setup','running','finished')),
  phase                 text not null default 'SETUP',   -- state-machine node; advances only on judge confirmation (D10)
  day_number            int  not null default 0,
  is_night              boolean not null default false,
  ruleset_version       text not null default 'emf-2026-07-proposed',
  allow_split_3_at_9    boolean not null default false,
  overlay_token         uuid not null default gen_random_uuid(),  -- safe view: never roles
  roles_overlay_token   uuid not null default gen_random_uuid(),  -- roles view: always roles (D7)
  created_at            timestamptz not null default now()
);

-- The 10 seated players. Nicknames are free text for the MVP (no player accounts).
-- state is recomputed from the action tables below, but remains directly editable by the
-- judge as an override (D8); fouls is a plain judge-edited counter (scope §3).
create table seats (
  id           uuid primary key default gen_random_uuid(),
  game_id      uuid not null references games(id) on delete cascade,
  seat_number  int  not null check (seat_number between 1 and 10),
  nickname     text not null,
  state        text not null default 'alive'
                 check (state in ('alive','eliminated','killed')),
  fouls        int  not null default 0,
  unique (game_id, seat_number)
);

-- THE SECRET. A separate table so it can be locked down independently of everything else.
create table game_roles (
  game_id      uuid not null references games(id) on delete cascade,
  seat_number  int  not null check (seat_number between 1 and 10),
  role         text not null check (role in ('civilian','mafia','don','sheriff')),
  primary key (game_id, seat_number)
);

-- ── Typed action records. All editable by the owning judge/admin (D8). ──

create table nominations (
  id           uuid primary key default gen_random_uuid(),
  game_id      uuid not null references games(id) on delete cascade,
  day_number   int  not null,
  order_no     int  not null,   -- nomination order; also final-word order (OQ-A)
  nominator    int  not null check (nominator between 1 and 10),
  nominee      int  not null check (nominee between 1 and 10),
  unique (game_id, day_number, order_no)
);

create table vote_rounds (
  id           uuid primary key default gen_random_uuid(),
  game_id      uuid not null references games(id) on delete cascade,
  day_number   int  not null,
  round_no     int  not null,   -- 1, 2 = tie re-vote, 3 = all-out
  unique (game_id, day_number, round_no)
);

-- D6: aggregate tallies per candidate. No voter identity.
create table vote_tallies (
  round_id       uuid not null references vote_rounds(id) on delete cascade,
  candidate_seat int  not null check (candidate_seat between 1 and 10),
  votes          int  not null check (votes >= 0),
  primary key (round_id, candidate_seat)
);

-- Elimination is a set (scope §6): one row per eliminated seat.
create table day_eliminations (
  game_id      uuid not null references games(id) on delete cascade,
  day_number   int  not null,
  seat_number  int  not null check (seat_number between 1 and 10),
  reason       text not null check (reason in ('vote','all_out_vote')),
  primary key (game_id, day_number, seat_number)
);

create table night_actions (
  game_id      uuid not null references games(id) on delete cascade,
  night_number int  not null,
  action       text not null check (action in ('mafia_shot','don_check','sheriff_check')),
  target_seat  int  check (target_seat between 1 and 10),   -- null = miss / no action
  primary key (game_id, night_number, action)
);

-- Best move: the first-killed player's up-to-three suspects.
create table best_move (
  game_id      uuid primary key references games(id) on delete cascade,
  seats        int[] not null check (array_length(seats, 1) <= 3)
);
```

**Roles in their own table** is the key structural choice. Postgres RLS is row-level, not
column-level — isolating the secret in its own table lets one tight policy govern all access
to it, rather than trying to hide a column on a table the overlay otherwise needs.

### Corrections and derived state (D8, D10)

The correction model in one line: **fix the fact, derived state follows.** "Sheriff checked
3, judge pressed 4" is fixed by editing that `night_actions` row — even several actions
later, with nothing to unwind. On any insert/update/delete of an action record, a small
`recompute_game(game_id)` function re-derives each seat's alive/dead state (from
`day_eliminations` and `mafia_shot` rows) and re-evaluates the win condition, in the same
transaction as the edit.

Two judge-authority escape hatches sit on top:

- The judge can **directly edit** a seat's `state`, `role`, or `fouls` (and any other
  record). A direct state edit is the final word until the next action-record edit triggers
  a recompute over it.
- The state machine never advances itself (D10): `games.phase` moves only when the judge
  confirms the proposed transition, so a mistaken entry can't cascade the game forward.

---

## 3. The security boundary (RLS)

This is the whole role-leakage defense. Enable RLS everywhere, then grant judges full access
to their own games, admins full access to every game (D9), and the overlay **nothing** on
`game_roles`.

```sql
alter table profiles         enable row level security;
alter table games            enable row level security;
alter table seats            enable row level security;
alter table game_roles       enable row level security;
alter table nominations      enable row level security;
alter table vote_rounds      enable row level security;
alter table vote_tallies     enable row level security;
alter table day_eliminations enable row level security;
alter table night_actions    enable row level security;
alter table best_move        enable row level security;

-- Everyone can read their own profile; role changes are an operator task, not app UI.
create policy own_profile on profiles for select using (auth.uid() = id);

-- Judge owns their games; admins access all games.
create policy staff_games on games
  for all using (auth.uid() = judge_id or is_admin())
          with check (auth.uid() = judge_id or is_admin());

-- Child tables: access follows the parent game. Shown for seats; nominations, vote_rounds,
-- day_eliminations, night_actions and best_move get the identical policy, and vote_tallies
-- checks via its parent vote_round's game.
create policy staff_seats on seats
  for all using  (auth.uid() = (select judge_id from games g where g.id = seats.game_id) or is_admin())
          with check (auth.uid() = (select judge_id from games g where g.id = seats.game_id) or is_admin());

-- Roles: readable/writable ONLY by the owning judge or an admin. There is deliberately NO
-- policy granting anon any access, so the public overlay key can never SELECT this table.
create policy staff_roles on game_roles
  for all using  (auth.uid() = (select judge_id from games g where g.id = game_roles.game_id) or is_admin())
          with check (auth.uid() = (select judge_id from games g where g.id = game_roles.game_id) or is_admin());
```

That's the entire boundary. Because no policy exposes `game_roles` to the anon key, a
curious viewer who extracts the anon key from the overlay page still cannot query roles.

---

## 4. How the overlay gets data without touching roles

Both views show seat numbers, nicknames, alive/dead, day/night — no speech timer (timing
lives on the judge's screen only). The roles view additionally shows every seat's role —
dead seats included — for the whole game; the safe view shows none, ever. Path:

1. **Initial render.** `GET /overlay/<token>` is a Next.js server component. It uses the
   Supabase **service role** (server-side only, never shipped to the browser) to look the
   token up against both token columns; which column matched decides whether roles are
   included. The browser receives already-filtered HTML.
2. **Live updates.** The overlay browser joins a Realtime **Broadcast** channel
   `overlay:<token>` (one channel per view) using the anon key. The running judge's (or
   admin's) client publishes a state snapshot on every change: a role-free payload to the
   safe channel, a full payload to the roles channel. Because the payload is constructed by
   trusted code, the boundary holds at construction time.

**Channel write-protection (O5).** A plain public Broadcast channel lets any subscriber also
*send* — anyone holding an overlay URL could publish spoofed state to the stream. Use
Realtime **private channels** with authorization policies on `realtime.messages`: anon may
only *receive* on its topic; only the owning judge or an admin may *send*.

Broadcast (not Postgres Changes) is chosen for the overlay on purpose: Postgres Changes
respects RLS by *granting the subscriber read access to the changed rows*, which would mean
giving the anon overlay read access to game state — reintroducing the leak. Broadcast is
pub/sub where the **sender** decides the payload, so filtering stays in trusted hands.

The judge panel, by contrast, can safely use Postgres Changes — it's authenticated and RLS
already grants it everything for its own game.

---

## 5. Setup flow (D1 — judge records the physical draw)

Players draw physical cards; the judge inputs what each seat drew. So role assignment is a
plain form the judge fills in during `CARD_DISTRIBUTION`, writing one `game_roles` row per
seat. The app does not randomize or deal, and the rows stay editable (D8) in case of
mis-entry.

---

## 6. Open items to validate before/while building

1. **Overlay liveness depends on the running client being connected** to broadcast. That's
   true in practice (the judge is running the game), but if their tab closes, the overlay
   freezes until reconnect — or until an admin opens the game and resumes broadcasting (D9).
   Acceptable for MVP; note it. A DB-trigger→Realtime alternative exists but adds DB logic
   against the minimal-maintenance goal.
2. **Recompute atomicity.** An action-record edit and `recompute_game` must run in one
   transaction — either a trigger on the action tables or a single write-RPC per edit.
   Decide at implementation; a trigger is less app code but more DB logic.
3. **OQ-A (final words on multi-elimination)** — resolved in scope §8: `FINAL_WORD_DAY`
   loops over the eliminated set, one 60s speech each, in nomination order (order still
   assumed, to confirm with client). Flow only; schema unaffected.
4. Service-role usage is confined to the overlay's server render. Keep the service key on the
   server; it must never reach any browser.
