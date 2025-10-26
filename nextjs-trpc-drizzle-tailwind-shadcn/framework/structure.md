# Project Structure

Define clear homes for code to keep boundaries strict and files minimal. Follow this structure unless a documented exception applies.

## Top‑Level Layout

- app/: Next.js App Router routes, layouts, and route handlers
- src/lib/api/: tRPC setup, routers, and command modules
- src/lib/db/: Drizzle schema and DB client
- src/components/ui/: shadcn/ui primitives and shared UI utilities
- src/components/features/: feature modules (containers + presentational)
- src/lib/types/ui/: UI-facing types derived from RouterOutputs
- src/styles/: global CSS (Tailwind v4 import, tokens, layers). Use `src/styles/global.css`.
- tests/: unit, integration/contracts, and e2e

## App Router

- Each route folder owns `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx` as needed
- Use route groups `(group)` to organize without affecting the URL
- Dynamic segments use `[param]`; `params` and `searchParams` are Promises in Server Components

```tsx
// app/(app)/tasks/page.tsx
import TasksClient from '@/components/features/tasks/TasksClient'

export default async function Page() {
  return <TasksClient />
}
```

## API Layer

- Routers live in `src/lib/api/routers/*`
- Commands live in `src/lib/api/commands/{queries|mutations}/…`
- Map input/output at the router; keep commands pure and typed
- Use a centralized error mapper; do not inline TRPC errors

```ts
// src/lib/api/routers/tasks.router.ts
export const tasksRouter = createTRPCRouter({
  list: rbacProcedure.input(ListTasksInputSchema).output(ListTasksOutputSchema).query(/* … */),
})
```

## Database

- Drizzle schema in `src/lib/db/schema/**`
- Migrations in `migrations/`
- Keep table names snake_case and columns snake_case

```ts
// src/lib/db/schema/tasks.ts
export const tasks = pgTable('tasks', { /* … */ })
```

## Features

- Feature folder: `src/components/features/<domain>/`
- Containers end with `*Client.tsx` and can use hooks
- Presentational components are props‑only and reusable
- UI types derive from RouterOutputs and live under `src/lib/types/ui/*`

```txt
src/components/features/tasks/
- TasksClient.tsx
- TaskList.tsx
- TaskForm.tsx
```

## Decision Tree

- Is it a page, layout, or route glue? Put it in app/
- Is it a server API contract? Router in `src/lib/api/routers/*`
- Is it domain logic for read/write? Command in `src/lib/api/commands/*`
- Is it DB structure? Drizzle schema in `src/lib/db/schema/*`
- Is it interactive UI for a feature? `src/components/features/<domain>/*`
- Is it a reusable UI primitive? `src/components/ui/*`
- Is it a UI type? `src/lib/types/ui/<feature>.ts`

When unsure, prefer commands for data logic, routers for validation and delegation, and pages for composition. Boundary and pattern checks are enforced via `make qa`.
