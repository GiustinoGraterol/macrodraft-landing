# Architecture

LoL Fantasy is a cross-platform app (iOS, Android, and web from one codebase)
for League of Legends esports stats and fantasy. The stack is fixed by **ADR-001
(GRA-11)** — do not introduce alternatives (Flutter, Next.js, Express/NestJS,
Firebase, Auth0, Socket.io, Redis…) without reopening the ADR.

## Stack

| Layer                          | Choice                                                 |
| ------------------------------ | ------------------------------------------------------ |
| Mobile (iOS/Android)           | React Native + Expo                                    |
| Web                            | The same RN app via React Native Web (Expo Web)        |
| DB / Auth / Backend / Realtime | Supabase (Postgres + Auth + Realtime + Edge Functions) |
| Server state                   | TanStack Query (React Query v5)                        |
| Styling                        | Tailwind via NativeWind                                |
| Language                       | TypeScript everywhere                                  |

There is **no separate web frontend and no custom backend server**: the Expo app
renders on web too, and Supabase is the backend. Realtime (live draft and
scoring) uses Supabase Realtime over WebSockets — no self-hosted socket server.

## Monorepo layout

```
.
├── apps/
│   └── mobile/          # Expo RN app (mobile + web). Added in GRA-16.
├── packages/
│   └── shared/          # Framework-free domain types & constants (TS)
├── supabase/
│   ├── config.toml      # Local Supabase stack config
│   ├── migrations/      # SQL migrations (source of truth for the schema)
│   ├── functions/       # Edge Functions (added as ingestion lands)
│   └── seed.sql         # Local dev seed data
├── .github/workflows/   # CI (lint, format, typecheck, test, build)
├── tsconfig.base.json   # Shared TS compiler options
└── package.json         # npm workspaces root
```

`@lol-fantasy/shared` is intentionally free of React/React Native so it can be
imported from both the app and the Supabase Edge Functions (Deno).

## Conventions

- **Routing:** Expo Router (file-based) so mobile and web share navigation.
- **Data access:** all Supabase access goes through TanStack Query hooks — no raw
  `supabase-js` calls scattered through components.
- **Security:** RLS enabled on every table. The client only ever holds the
  `anon` key; the `service_role` key is server/ingestion only and never shipped.
- **Database:** migrations in `supabase/migrations/` are the source of truth;
  apply locally with `npm run db:reset`.

## Local development

```bash
npm install            # install workspace deps
npm run db:start       # start local Supabase (requires Docker + Supabase CLI)
npm run db:reset       # apply migrations + seed
npm run lint && npm run typecheck && npm run test && npm run build
```

## External data sources (approved, respect ToS / rate limits)

- Riot API (GRA-5), Data Dragon (GRA-6), Leaguepedia Cargo (GRA-8),
  patch notes (GRA-10).
