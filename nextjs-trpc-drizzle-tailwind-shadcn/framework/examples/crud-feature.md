# CRUD Feature

A complete CRUD flow for `tasks` with schema, router, UI list/create/update/delete, validation, and optimistic updates.

## Schema

```ts
// src/lib/db/schema/tasks.ts
import { pgTable, serial, text, boolean, timestamp, index } from 'drizzle-orm/pg-core'
import type { InferSelectModel, InferInsertModel } from 'drizzle-orm'

export const tasks = pgTable(
  'tasks',
  {
    id: serial('id').primaryKey(),
    title: text('title').notNull(),
    done: boolean('done').notNull().default(false),
    created_at: timestamp('created_at', { withTimezone: true })
      .notNull()
      .defaultNow(),
    updated_at: timestamp('updated_at', { withTimezone: true })
      .notNull()
      .defaultNow()
      .$onUpdate(() => new Date()),
  },
  (t) => [
    index('tasks_done_idx').on(t.done),
    index('tasks_created_at_idx').on(t.created_at),
  ]
)

// Export type-safe models
export type Task = InferSelectModel<typeof tasks>
export type NewTask = InferInsertModel<typeof tasks>
```

## Router

```ts
// src/lib/api/routers/tasks.router.ts
import { z } from 'zod'
import { createTRPCRouter, rbacProcedure, mapAppError } from '../trpc'
import {
  ListTasksInputSchema,
  ListTasksOutputSchema,
  listTasksQuery,
} from '@/lib/api/commands/queries/tasks/list-tasks.query'
import {
  CreateTaskInputSchema,
  CreateTaskOutputSchema,
  createTaskMutation,
} from '@/lib/api/commands/mutations/tasks/create-task.mutation'
import {
  UpdateTaskInputSchema,
  UpdateTaskOutputSchema,
  updateTaskMutation,
} from '@/lib/api/commands/mutations/tasks/update-task.mutation'
import {
  DeleteTaskInputSchema,
  DeleteTaskOutputSchema,
  deleteTaskMutation,
} from '@/lib/api/commands/mutations/tasks/delete-task.mutation'

export const tasksRouter = createTRPCRouter({
  list: rbacProcedure
    .input(ListTasksInputSchema.strict())
    .output(ListTasksOutputSchema.strict())
    .query(async ({ ctx, input }) => {
      try {
        return await listTasksQuery(ctx, input)
      } catch (e) {
        mapAppError(e)
      }
    }),

  create: rbacProcedure
    .input(CreateTaskInputSchema.strict())
    .output(CreateTaskOutputSchema.strict())
    .mutation(async ({ ctx, input }) => {
      try {
        return await createTaskMutation(ctx, input)
      } catch (e) {
        mapAppError(e)
      }
    }),

  update: rbacProcedure
    .input(UpdateTaskInputSchema.strict())
    .output(UpdateTaskOutputSchema.strict())
    .mutation(async ({ ctx, input }) => {
      try {
        return await updateTaskMutation(ctx, input)
      } catch (e) {
        mapAppError(e)
      }
    }),

  delete: rbacProcedure
    .input(DeleteTaskInputSchema.strict())
    .output(DeleteTaskOutputSchema.strict())
    .mutation(async ({ ctx, input }) => {
      try {
        return await deleteTaskMutation(ctx, input)
      } catch (e) {
        mapAppError(e)
      }
    }),
})

// Export router type for client usage
export type TasksRouter = typeof tasksRouter
```

## UI

```tsx
// src/components/features/tasks/TasksClient.tsx
'use client'

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { trpc, type RouterOutputs, type RouterInputs } from '@/lib/api/react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { useState, useCallback, type FormEvent } from 'react'

// Derive types from router
type TasksList = RouterOutputs['tasks']['list']
type Task = TasksList['items'][number]
type CreateTaskInput = RouterInputs['tasks']['create']
type DeleteTaskInput = RouterInputs['tasks']['delete']

export default function TasksClient() {
  const qc = useQueryClient()
  const [title, setTitle] = useState('')

  const listQ = trpc.tasks.list.queryOptions({ page: 1, pageSize: 50 })
  const list = useQuery(listQ)

  const create = useMutation({
    ...trpc.tasks.create.mutationOptions(),
    onMutate: async (vars: CreateTaskInput) => {
      // Cancel outgoing refetches
      await qc.cancelQueries({ queryKey: listQ.queryKey })

      // Snapshot previous value
      const prev = qc.getQueryData<TasksList>(listQ.queryKey)

      // Optimistically update
      qc.setQueryData<TasksList>(listQ.queryKey, (old) => {
        if (!old) return old

        const optimisticTask: Task = {
          id: -Date.now(),
          title: vars.title,
          done: false,
        }

        return {
          ...old,
          items: [optimisticTask, ...old.items],
          totalCount: old.totalCount + 1,
        }
      })

      return { prev }
    },
    onError: (_err, _vars, ctx) => {
      // Rollback on error
      if (ctx?.prev) {
        qc.setQueryData<TasksList>(listQ.queryKey, ctx.prev)
      }
    },
    onSettled: () => {
      // Refetch after mutation
      qc.invalidateQueries({ queryKey: listQ.queryKey })
    },
  })

  const remove = useMutation({
    ...trpc.tasks.delete.mutationOptions(),
    onMutate: async (vars: DeleteTaskInput) => {
      await qc.cancelQueries({ queryKey: listQ.queryKey })

      const prev = qc.getQueryData<TasksList>(listQ.queryKey)

      qc.setQueryData<TasksList>(listQ.queryKey, (old) => {
        if (!old) return old

        return {
          ...old,
          items: old.items.filter((t) => t.id !== vars.id),
          totalCount: old.totalCount - 1,
        }
      })

      return { prev }
    },
    onError: (_err, _vars, ctx) => {
      if (ctx?.prev) {
        qc.setQueryData<TasksList>(listQ.queryKey, ctx.prev)
      }
    },
    onSettled: () => {
      qc.invalidateQueries({ queryKey: listQ.queryKey })
    },
  })

  const handleSubmit = useCallback(
    async (e: FormEvent<HTMLFormElement>) => {
      e.preventDefault()
      if (!title.trim()) return

      await create.mutateAsync({ title: title.trim() })
      setTitle('')
    },
    [title, create]
  )

  const handleDelete = useCallback(
    (id: number) => {
      remove.mutate({ id })
    },
    [remove]
  )

  if (list.isPending) {
    return <div>Loading tasks...</div>
  }

  if (list.error) {
    return <div>Error: {list.error.message}</div>
  }

  return (
    <div className="space-y-4">
      <form onSubmit={handleSubmit} className="flex gap-2">
        <Input
          value={title}
          onChange={(e) => setTitle(e.target.value)}
          placeholder="New task"
          disabled={create.isPending}
        />
        <Button type="submit" disabled={create.isPending || !title.trim()}>
          {create.isPending ? 'Adding...' : 'Add'}
        </Button>
      </form>

      <div>
        <p className="text-sm text-muted-foreground mb-2">
          {list.data.totalCount} {list.data.totalCount === 1 ? 'task' : 'tasks'}
        </p>
        <ul className="space-y-2">
          {list.data.items.map((task) => (
            <li key={task.id} className="flex items-center justify-between">
              <span className={task.done ? 'line-through' : ''}>{task.title}</span>
              <Button
                variant="secondary"
                size="sm"
                onClick={() => handleDelete(task.id)}
                disabled={remove.isPending}
              >
                Delete
              </Button>
            </li>
          ))}
        </ul>
      </div>
    </div>
  )
}
```

## Verification

- Creating and deleting update the list optimistically
- Types propagate from router to client
- List output stays normalized

## Related

- ../patterns/queries.md
- ../patterns/mutations.md
- ../patterns/shadcn.md
