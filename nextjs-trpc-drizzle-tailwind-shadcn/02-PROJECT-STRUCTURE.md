# Project Structure

## Directory Rules
`src/`
- `app/`              — Next.js App Router ONLY
- `components/`       — Shared components ONLY
- `server/`           — Backend logic ONLY
  - `api/`           — tRPC routers
  - `db/`            — Drizzle schemas & queries
- `lib/`              — Utilities ONLY (pure, framework-agnostic)

## File Placement Rules
- IF component used in 1 place → co-locate under that route folder
- IF component used in 2+ places → `src/components/`
- IF logic touches DB or secrets → `src/server/`
- IF network boundary (webhook, tRPC endpoint) → `app/api/*`

## Decision Tree
1) Does it render UI?
   - Yes → Component. Co-locate if single-use, else `components/`.
2) Does it read/write the DB?
   - Yes → `server/db/*` (schemas, queries). Expose only via tRPC.
3) Is it an API boundary?
   - Yes → Route Handler at `app/api/.../route.ts`.
4) Is it shared utility with no framework dependencies?
   - Yes → `lib/`.

## Minimal Example
```
src/
  app/
    posts/
      page.tsx                # Server Component
    api/
      trpc/[trpc]/route.ts    # tRPC route handler (fetch adapter)
  server/
    api/
      routers/
        posts.ts              # tRPC router for posts
        _app.ts               # merges routers
    db/
      schema/
        posts.ts              # Drizzle table + relations
      index.ts                # db client setup
  components/
    PostCard.tsx              # Reusable component
  lib/
    formatDate.ts             # Pure util
```

