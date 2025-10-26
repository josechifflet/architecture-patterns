# Framework Principles

This framework reduces decisions by repeating a small set of patterns. Every feature follows the same path from DB to UI with clear boundaries and typed contracts.

## Minimal Files

- Co‑locate at the feature level and use purposeful files only
- Pages stay thin; data lives in commands; routers validate and delegate

Example: a page renders a container; the container calls tRPC; the command talks to the DB.

```tsx
// app/tasks/page.tsx (Server Component)
import TasksClient from '@/components/features/tasks/TasksClient'

export default async function Page() {
  // Keep orchestration only; containers do the fetching
  return <TasksClient />
}
```

For Server Components that read data, prefer a server caller:

```ts
import { api } from '@/lib/api/server'

export default async function Page() {
  const caller = await api()
  const data = await caller.tasks.list({ page: 1, pageSize: 20 })
  return <pre>{JSON.stringify(data, null, 2)}</pre>
}
```

## One Pattern Per Problem

- Queries follow a normalized list shape
- Mutations own invariants and transactions
- Routers are thin with input/output schemas

```ts
// src/lib/api/commands/queries/tasks/list-tasks.query.ts
import { z } from 'zod'

export const ListTasksInputSchema = z.object({
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(20),
  search: z.string().optional(),
}).strict()

export const ListTasksOutputSchema = z.object({
  items: z.array(z.object({ id: z.number(), title: z.string(), done: z.boolean() })),
  page: z.number(),
  pageSize: z.number(),
  totalCount: z.number(),
  totalPages: z.number(),
})

export type ListTasksInput = z.infer<typeof ListTasksInputSchema>
export type ListTasksOutput = z.infer<typeof ListTasksOutputSchema>
```

## Repetition Over Abstraction

- Prefer duplicating a simple, correct pattern over inventing abstractions
- Keep files predictable so new engineers can copy a known shape

```ts
// src/lib/api/routers/tasks.router.ts
import { z } from 'zod'
import { createTRPCRouter, rbacProcedure } from '../trpc'
import { ListTasksInputSchema, ListTasksOutputSchema } from '../commands/queries/tasks/list-tasks.query'
import { listTasks } from '../commands/queries/tasks/list-tasks.impl'

export const tasksRouter = createTRPCRouter({
  list: rbacProcedure
    .input(ListTasksInputSchema)
    .output(ListTasksOutputSchema)
    .query(async ({ ctx, input }) => listTasks(ctx, input)),
})
```

## Type Safety First

- End‑to‑end types: infer input/output from Zod and router types
- Database types: Drizzle infers column and row types from schema
- React Query types: tRPC v11 exposes type‑safe `queryOptions` and `mutationOptions`

```ts
// src/lib/db/schema/tasks.ts
import { pgTable, serial, text, boolean, index } from 'drizzle-orm/pg-core'

export const tasks = pgTable(
  'tasks',
  {
    id: serial('id').primaryKey(),
    title: text('title').notNull(),
    done: boolean('done').notNull().default(false),
  },
  (table) => ({ titleIdx: index('tasks_title_idx').on(table.title) })
)
```

```ts
// src/components/features/tasks/TasksClient.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { trpc } from '@/lib/api/react'

export default function TasksClient() {
  const list = useQuery(trpc.tasks.list.queryOptions({ page: 1, pageSize: 20 }))
  if (list.isPending) return <div>Loading…</div>
  if (list.error) return <div>Failed to load</div>
  return (
    <ul>
      {list.data.items.map((t) => (
        <li key={t.id}>{t.title}</li>
      ))}
    </ul>
  )
}
```

For cache updates, let clients or route handlers decide which query keys or cache tags to invalidate. Keep command outputs focused on domain data.

Use these principles to choose defaults. If unsure, pick the simplest option that preserves types and matches the patterns.
