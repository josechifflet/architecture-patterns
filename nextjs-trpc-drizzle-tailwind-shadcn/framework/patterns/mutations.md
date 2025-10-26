# Mutations

Mutations change server state. Keep invariants and transactions in commands. Validate at the router boundary and return typed results. Commands may return invalidation hints.

## When To Use

- Creating, updating, or deleting records
- Enforcing invariants and permissions
- Returning invalidation hints or tags

## Basic Structure

```ts
// src/lib/api/commands/mutations/tasks/toggle-task.mutation.ts
import { z } from 'zod'
import { db } from '@/lib/db'
import { tasks } from '@/lib/db/schema/tasks'
import { eq } from 'drizzle-orm'

// Input schema with strict validation
export const ToggleTaskInputSchema = z
  .object({
    id: z.number().int().positive(),
  })
  .strict()

// Output schema with explicit structure
export const ToggleTaskOutputSchema = z
  .object({
    id: z.number().int().positive(),
    done: z.boolean(),
  })
  .strict()

// Export inferred types for type safety
export type ToggleTaskInput = z.infer<typeof ToggleTaskInputSchema>
export type ToggleTaskOutput = z.infer<typeof ToggleTaskOutputSchema>

// Type-safe context (requires authenticated user)
type MutationContext = {
  readonly userId: string
  readonly permissions: readonly string[]
}

export async function toggleTask(
  ctx: MutationContext,
  input: ToggleTaskInput
): Promise<ToggleTaskOutput> {
  // Fetch current state
  const [current] = await db
    .select({ done: tasks.done })
    .from(tasks)
    .where(eq(tasks.id, input.id))
    .limit(1)

  if (!current) {
    throw { code: 'NOT_FOUND', message: `Task ${input.id} not found` }
  }

  // Toggle the done state
  const [row] = await db
    .update(tasks)
    .set({ done: !current.done })
    .where(eq(tasks.id, input.id))
    .returning({ id: tasks.id, done: tasks.done })

  if (!row) {
    throw { code: 'INTERNAL', message: 'Failed to update task' }
  }

  return row
}
```

## Complete Example

```ts
// src/lib/api/routers/tasks.router.ts
import { createTRPCRouter, rbacProcedure, mapAppError } from '../trpc'
import {
  ToggleTaskInputSchema,
  ToggleTaskOutputSchema,
  toggleTask,
  type ToggleTaskOutput,
} from '../commands/mutations/tasks/toggle-task.mutation'

export const tasksRouter = createTRPCRouter({
  toggle: rbacProcedure
    .input(ToggleTaskInputSchema.strict())
    .output(ToggleTaskOutputSchema.strict())
    .mutation(async ({ ctx, input }) => {
      try {
        return await toggleTask(ctx, input)
      } catch (e) {
        mapAppError(e)
      }
    }),
})

// Export router type for client usage
export type TasksRouter = typeof tasksRouter
```

```tsx
// src/components/features/tasks/TaskActionsClient.tsx
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'
import { trpc, type RouterOutputs } from '@/lib/api/react'
import { Button } from '@/components/ui/button'

type ToggleTaskOutput = RouterOutputs['tasks']['toggle']

interface TaskActionsClientProps {
  readonly id: number
}

export function TaskActionsClient({ id }: TaskActionsClientProps) {
  const qc = useQueryClient()

  const toggle = useMutation({
    ...trpc.tasks.toggle.mutationOptions(),
    onSuccess: (data: ToggleTaskOutput) => {
      // Invalidate list query to refetch
      qc.invalidateQueries({ queryKey: trpc.tasks.list.queryKey() })

      // Optionally update cache optimistically
      console.log(`Task ${data.id} toggled to ${data.done}`)
    },
    onError: (error) => {
      console.error('Failed to toggle task:', error.message)
    },
  })

  return (
    <Button
      onClick={() => toggle.mutate({ id })}
      disabled={toggle.isPending}
      variant="secondary"
    >
      {toggle.isPending ? 'Toggling...' : 'Toggle'}
    </Button>
  )
}
```

## Common Mistakes

- Handling transactions outside commands
- Throwing raw errors instead of mapping to safe TRPC codes
- Returning inconsistent shapes across mutations

## How To

1. Define input/output Zod schemas colocated with the mutation command.
2. Implement the DB write in the command; enforce invariants/permissions.
3. Wire the router with `.input().output().mutation()` delegating to the command.
4. In clients, use `useMutation` and invalidate related query keys on success.

## Anti-pattern

- Doing writes inside routers instead of commands
- Returning ad-hoc shapes per mutation
- Forgetting to invalidate affected lists/details

## Related

- forms.md
- trpc-procedures.md
