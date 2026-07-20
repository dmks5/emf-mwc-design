# EMF / MWC Mafia Judging Website — Design Documents

Public design documents for a tournament-Mafia judging application built for the
**English Mafia Federation (EMF)** and **Mafia World Cup (MWC)**. A judge runs a live
10-player Mafia game end to end; a separate read-only overlay renders the game state for a
stream audience.

## Contents

- **[mvp-scope.md](./mvp-scope.md)** — the first deliverable, scoped: complete one game plus
  a live overlay. Locked decisions, game settings, open questions.
- **[technical-design.md](./technical-design.md)** — data model, the role-security boundary
  (Row-Level Security), and how the overlay stays live without leaking roles.
- **[game-rules.md](./game-rules.md)** — implementable game rules and the canonical game-flow
  state machine. An unofficial working extraction — see the disclaimer at the top of the file.

## Stack

Next.js (App Router) on Vercel · Supabase Cloud (Postgres, Realtime, RLS) · Supabase Auth.

## Note

These are design documents only. They reference internal requirements and official-rules
documents that are not part of this public repository.
