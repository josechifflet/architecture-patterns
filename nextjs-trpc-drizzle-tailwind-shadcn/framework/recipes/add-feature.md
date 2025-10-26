# Add Feature

Create a complete feature from database to UI using the Tasks domain as an example.

## Prerequisites

- Database configured and migrations working
- tRPC server and fetch adapter mounted under `/api/trpc`
- Tailwind v4 global CSS imported
- shadcn/ui installed

## Steps

1. Create Drizzle schema

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

2. Add list query

```ts
// src/lib/api/commands/queries/tasks/list-tasks.query.ts
import { z } from 'zod'
import { db } from '@/lib/db'
import { tasks } from '@/lib/db/schema/tasks'

export const ListTasksInputSchema = z.object({ page: z.number().int().min(1).default(1), pageSize: z.number().int().min(1).max(100).default(20) }).strict()
export const ListTasksOutputSchema = z.object({ items: z.array(z.object({ id: z.number(), title: z.string(), done: z.boolean() })), page: z.number(), pageSize: z.number(), totalCount: z.number(), totalPages: z.number() })

export async function listTasks(_ctx: { userId: string }, { page, pageSize }: z.infer<typeof ListTasksInputSchema>) {
  const [{ count }] = await db.select({ count: db.fn.count(tasks.id) }).from(tasks)
  const items = await db.select({ id: tasks.id, title: tasks.title, done: tasks.done }).from(tasks).limit(pageSize).offset((page - 1) * pageSize)
  const totalCount = Number(count)
  const totalPages = Math.max(1, Math.ceil(totalCount / pageSize))
  return { items, page, pageSize, totalCount, totalPages }
}
```

3. Add create mutation

```ts
// src/lib/api/commands/mutations/tasks/create-task.mutation.ts
import { z } from 'zod'
import { db } from '@/lib/db'
import { tasks } from '@/lib/db/schema/tasks'

export const CreateTaskInputSchema = z.object({ title: z.string().min(1) }).strict()
export const CreateTaskOutputSchema = z.object({ id: z.number() })

export async function createTask(_ctx: { userId: string }, input: z.infer<typeof CreateTaskInputSchema>) {
  const [row] = await db.insert(tasks).values({ title: input.title }).returning({ id: tasks.id })
  return { id: row.id }
}
```

4. Wire the router

```ts
// src/lib/api/routers/tasks.router.ts
import { createTRPCRouter, rbacProcedure } from '../trpc'
import { ListTasksInputSchema, ListTasksOutputSchema, listTasks } from '../commands/queries/tasks/list-tasks.query'
import { CreateTaskInputSchema, CreateTaskOutputSchema, createTask } from '../commands/mutations/tasks/create-task.mutation'

export const tasksRouter = createTRPCRouter({
  list: rbacProcedure.input(ListTasksInputSchema).output(ListTasksOutputSchema).query(({ ctx, input }) => listTasks(ctx, input)),
  create: rbacProcedure.input(CreateTaskInputSchema).output(CreateTaskOutputSchema).mutation(({ ctx, input }) => createTask(ctx, input)),
})
```

5. Build UI

```tsx
// src/components/features/tasks/TasksClient.tsx
'use client'

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { trpc } from '@/lib/api/react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { useTranslations } from 'next-intl'

export default function TasksClient() {
  const qc = useQueryClient()
  const t = useTranslations('Tasks')
  const list = useQuery(trpc.tasks.list.queryOptions({ page: 1, pageSize: 20 }))
  const create = useMutation({
    ...trpc.tasks.create.mutationOptions(),
    onSuccess: () => qc.invalidateQueries({ queryKey: trpc.tasks.list.queryKey() }),
  })

  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    const fd = new FormData(e.currentTarget)
    const title = String(fd.get('title') ?? '')
    if (!title) return
    await create.mutateAsync({ title })
    e.currentTarget.reset()
  }

  if (list.isPending) return <div>{t('loading')}</div>
  if (list.error) return <div>{t('loadError')}</div>

  return (
    <div className="space-y-4">
      <form onSubmit={onSubmit} className="flex gap-2">
        <Input name="title" placeholder={t('newTaskPlaceholder')} />
        <Button type="submit" disabled={create.isPending}>{t('add')}</Button>
      </form>
      <ul>{list.data.items.map((t) => <li key={t.id}>{t.title}</li>)}</ul>
    </div>
  )
}
```

6. Page glue

```tsx
// app/(app)/tasks/page.tsx
import TasksClient from '@/components/features/tasks/TasksClient'

export default async function Page() {
  return <TasksClient />
}
```

7. Add messages

```json
// src/locales/en.json (excerpt)
{
  "Tasks": {
    "loading": "Loading…",
    "loadError": "Failed to load",
    "add": "Add",
    "newTaskPlaceholder": "New task"
  }
}
```

## Verification

- Visit `/tasks` and add a task
- Check the list updates and pagination shape
- Confirm types for query and mutation inputs/outputs
- Switch locale and verify UI strings change

## Related

- patterns/queries.md
- patterns/mutations.md
- patterns/trpc-procedures.md
- patterns/server-components.md
- patterns/server-i18n.md
