# Queries

Queries read data with a normalized list shape. Keep filtering and sorting explicit. Paginate with `page` and `pageSize` and return `totalCount` and `totalPages`.

## When To Use

- Listing, filtering, and sorting records
- Joining related data for read models
- Stable list responses for client caches

## Basic Structure

```ts
// src/lib/api/commands/queries/tasks/list-tasks.query.ts
import { z } from 'zod'
import { db } from '@/lib/db'
import { tasks } from '@/lib/db/schema/tasks'
import { and, desc, ilike, eq, count, type SQL } from 'drizzle-orm'
import type { PgColumn } from 'drizzle-orm/pg-core'

// Define allowed sort fields using const assertion for exhaustiveness
const TASK_SORT_FIELDS = ['created_at', 'title'] as const
type TaskSortField = (typeof TASK_SORT_FIELDS)[number]

// Input schema with strict validation
export const ListTasksInputSchema = z
  .object({
    page: z.number().int().positive().default(1),
    pageSize: z.number().int().positive().max(100).default(20),
    search: z.string().min(1).optional(),
    done: z.boolean().optional(),
    sort: z.enum(TASK_SORT_FIELDS).default('created_at'),
  })
  .strict()

// Define output item schema separately for reusability
const TaskItemSchema = z.object({
  id: z.number().int().positive(),
  title: z.string(),
  done: z.boolean(),
})

// Normalized list output with strict pagination contract
export const ListTasksOutputSchema = z
  .object({
    items: z.array(TaskItemSchema),
    page: z.number().int().positive(),
    pageSize: z.number().int().positive(),
    totalCount: z.number().int().nonnegative(),
    totalPages: z.number().int().positive(),
  })
  .strict()

// Export inferred types
export type ListTasksInput = z.infer<typeof ListTasksInputSchema>
export type ListTasksOutput = z.infer<typeof ListTasksOutputSchema>
export type TaskItem = z.infer<typeof TaskItemSchema>

// Type-safe context (requires authenticated user)
type QueryContext = {
  readonly userId: string
  readonly permissions: readonly string[]
}

export async function listTasks(
  _ctx: QueryContext,
  input: ListTasksInput
): Promise<ListTasksOutput> {
  const { page, pageSize, search, done, sort } = input

  // Build where clause with explicit typing
  const conditions: (SQL | undefined)[] = [
    search ? ilike(tasks.title, `%${search}%`) : undefined,
    done !== undefined ? eq(tasks.done, done) : undefined,
  ]
  const where = and(...conditions)

  // Get total count
  const [{ count: total }] = await db
    .select({ count: count(tasks.id) })
    .from(tasks)
    .where(where)

  // Type-safe sort column selection
  const sortColumn: PgColumn = sort === 'created_at' ? tasks.id : tasks.title

  // Fetch paginated items
  const items = await db
    .select({
      id: tasks.id,
      title: tasks.title,
      done: tasks.done,
    })
    .from(tasks)
    .where(where)
    .orderBy(desc(sortColumn))
    .limit(pageSize)
    .offset((page - 1) * pageSize)

  const totalCount = Number(total)
  const totalPages = Math.max(1, Math.ceil(totalCount / pageSize))

  return {
    items,
    page,
    pageSize,
    totalCount,
    totalPages,
  }
}
```

## Complete Example

```ts
// src/lib/api/routers/tasks.router.ts
import { createTRPCRouter, rbacProcedure } from '../trpc'
import {
  ListTasksInputSchema,
  ListTasksOutputSchema,
  listTasks,
  type ListTasksOutput,
} from '../commands/queries/tasks/list-tasks.query'

export const tasksRouter = createTRPCRouter({
  list: rbacProcedure
    .input(ListTasksInputSchema.strict())
    .output(ListTasksOutputSchema.strict())
    .query(async ({ ctx, input }) => {
      return listTasks(ctx, input)
    }),
})

// Export router type for client usage
export type TasksRouter = typeof tasksRouter
```

```tsx
// src/components/features/tasks/TasksClient.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { trpc, type RouterOutputs } from '@/lib/api/react'
import type { ListTasksOutput } from '@/lib/api/commands/queries/tasks/list-tasks.query'

type TasksList = RouterOutputs['tasks']['list']

export default function TasksClient() {
  const list = useQuery(
    trpc.tasks.list.queryOptions({
      page: 1,
      pageSize: 20,
    })
  )

  if (list.isPending) return <div>Loading…</div>
  if (list.error) return <div>Error: {list.error.message}</div>

  // list.data is fully typed as TasksList
  return (
    <div>
      <p>Total: {list.data.totalCount}</p>
      <p>Pages: {list.data.totalPages}</p>
    </div>
  )
}
```

## Common Mistakes

- Returning inconsistent shapes; keep list outputs normalized
- Hiding sorting behavior; make it explicit in input
- Filtering on the client when the DB can do it

## How To

1. Define `List<Input>Schema` with `page`, `pageSize`, filters, and `sort` enum.
2. Compute `where` conditions explicitly and run a `count` query for totals.
3. Select display fields only, apply `orderBy`, `limit`, `offset`.
4. Return `{ items, page, pageSize, totalCount, totalPages }`.

## Anti-pattern

- Returning arrays without pagination metadata
- Implicit sorting or relying on DB defaults
- Leaking raw table rows instead of shaped projections

## Related

- trpc-procedures.md
- server-components.md
