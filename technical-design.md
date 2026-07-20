# Technical Design — MVP (Complete a Game + Overlay)

**Stack:** Next.js (App Router) on Vercel · Supabase Cloud (Postgres, Realtime, RLS) ·
Supabase Auth. See [mvp-scope.md](./mvp-scope.md) for scope and locked decisions D1–D6.

This document covers the data model and the role-security boundary for the MVP only.
Game-flow logic (the 19-state machine) lives in TypeScript in the Next.js app and is
specified in [game-rules.md](./game-rules.md) §1 — not repeated here.

---

## 1. Surfaces and who talks to what

Two clients, one backend:

- **Judge control panel** — authenticated (Supabase Auth). Talks directly to Postgres via
  the Supabase client, constrained by RLS. Holds the game-flow logic; writes events.
- **Stream overlay** — unauthenticated, reached by an unguessable per-game URL token.
  **Never reads roles from the database.** Gets its initial state from the Next.js server
  (service role, roles pre-filtered) and live updates via a Realtime Broadcast channel whose
  payload the judge's client controls.

The asymmetry is deliberate: the judge is a trusted authenticated actor; the overlay is a
public surface and is treated as hostile by default.

---

## 2. Data model

Event-sourced (D4): the `events` table is the source of truth; `games`/`seats` carry the
current projection for convenient querying and Realtime.

```sql
-- One row per game, owned by the judge who created it.
create table games (
  id                    uuid primary key default gen_random_uuid(),
  judge_id              uuid not null references auth.users(id),
  status                text not null default 'setup'
                          check (status in ('setup','running','finished')),
  phase                 text not null default 'SETUP',   -- current state-machine node
  day_number            int  not null default 0,
  is_night              boolean not null default false,
  ruleset_version       text not null default 'emf-2026-07-proposed',
  allow_split_3_at_9    boolean not null default false,
  show_roles_on_overlay boolean not null default false,  -- D5: defaults OFF
  overlay_token         uuid not null default gen_random_uuid(),  -- unguessable overlay key
  created_at            timestamptz not null default now()
);

-- The 10 seated players. Nicknames are free text for the MVP (no player accounts).
create table seats (
  id           uuid primary key default gen_random_uuid(),
  game_id      uuid not null references games(id) on delete cascade,
  seat_number  int  not null check (seat_number between 1 and 10),
  nickname     text not null,
  state        text not null default 'alive'
                 check (state in ('alive','eliminated','killed')),
  unique (game_id, seat_number)
);

-- THE SECRET. A separate table so it can be locked down independently of everything else.
create table game_roles (
  game_id      uuid not null references games(id) on delete cascade,
  seat_number  int  not null check (seat_number between 1 and 10),
  role         text not null check (role in ('civilian','mafia','don','sheriff')),
  primary key (game_id, seat_number)
);

-- Append-only log of every judge action. Never updated or deleted.
create table events (
  id           bigint generated always as identity primary key,
  game_id      uuid not null references games(id) on delete cascade,
  seq          int  not null,                 -- per-game ordering
  type         text not null,                 -- 'ROLE_ASSIGNED','VOTE_ROUND','SHOT', ...
  payload      jsonb not null default '{}',
  created_at   timestamptz not null default now(),
  unique (game_id, seq)
);
```

**Roles in their own table** is the key structural choice. Postgres RLS is row-level, not
column-level — isolating the secret in its own table lets one tight policy govern all access
to it, rather than trying to hide a column on a table the overlay otherwise needs.

### Votes (D6 — aggregate)

A voting round is one event; the tally is in the payload. No per-voter rows.

```jsonc
{ "type": "VOTE_ROUND",
  "payload": { "round": 1, "tallies": [ {"seat": 3, "votes": 4}, {"seat": 7, "votes": 6} ] } }
```

Elimination is a **set** (§6 of scope) — the all-out vote can remove several seats:

```jsonc
{ "type": "DAY_ELIMINATION", "payload": { "seats": [3, 7], "reason": "all_out_vote" } }
```

---

## 3. The security boundary (RLS)

This is the whole role-leakage defense. Enable RLS everywhere, then grant the judge full
access to their own games and grant the overlay **nothing** on `game_roles`.

```sql
alter table games      enable row level security;
alter table seats      enable row level security;
alter table game_roles enable row level security;
alter table events     enable row level security;

-- Judge owns their games.
create policy judge_games on games
  for all using (auth.uid() = judge_id)
          with check (auth.uid() = judge_id);

-- Seats / events: access follows ownership of the parent game.
create policy judge_seats on seats
  for all using  (auth.uid() = (select judge_id from games g where g.id = seats.game_id))
          with check (auth.uid() = (select judge_id from games g where g.id = seats.game_id));

create policy judge_events on events
  for all using  (auth.uid() = (select judge_id from games g where g.id = events.game_id))
          with check (auth.uid() = (select judge_id from games g where g.id = events.game_id));

-- Roles: readable/writable ONLY by the owning judge. There is deliberately NO policy
-- granting anon any access, so the public overlay key can never SELECT this table.
create policy judge_roles on game_roles
  for all using  (auth.uid() = (select judge_id from games g where g.id = game_roles.game_id))
          with check (auth.uid() = (select judge_id from games g where g.id = game_roles.game_id));
```

That's the entire boundary. Because no policy exposes `game_roles` to the anon key, a
curious viewer who extracts the anon key from the overlay page still cannot query roles.

---

## 4. How the overlay gets data without touching roles

The overlay must show seat numbers, nicknames, alive/dead, day/night — and roles **only when
enabled, and never for eliminated players** (O4). Path:

1. **Initial render.** `GET /overlay/<overlay_token>` is a Next.js server component. It uses
   the Supabase **service role** (server-side only, never shipped to the browser) to load the
   game by token, then builds the view — including roles **only if** `show_roles_on_overlay`
   is true, and omitting the role of any non-alive seat. The browser receives already-filtered
   HTML.
2. **Live updates.** The overlay browser joins a Realtime **Broadcast** channel
   `overlay:<overlay_token>` using the anon key. The judge's client publishes a filtered state
   snapshot to that channel on every change. Roles are included in the payload only under the
   same two conditions. Because the payload is constructed by trusted code, the boundary holds
   at construction time.

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
seat plus a `ROLE_ASSIGNED` event. The app does not randomize or deal.

---

## 6. Open items to validate before/while building

1. **Overlay liveness depends on the judge's client being connected** to broadcast. That's
   true in practice (the judge is running the game), but if their tab closes, the overlay
   freezes until reconnect. Acceptable for MVP; note it. A DB-trigger→Realtime alternative
   exists but adds DB logic against the minimal-maintenance goal.
2. **`seq` generation.** Per-game sequence needs to be gap-free and race-free. Simplest:
   allocate `seq` in a small Postgres function that locks the game row, or derive from
   `count`+advisory lock. To decide at implementation.
3. **OQ-A (final words on multi-elimination)** — flow only, not schema. The state machine's
   `FINAL_WORD_DAY` either fires once or loops over the eliminated set; schema handles either.
   Still needs a client answer, but does not block the build.
4. Service-role usage is confined to the overlay's server render. Keep the service key on the
   server; it must never reach any browser.
