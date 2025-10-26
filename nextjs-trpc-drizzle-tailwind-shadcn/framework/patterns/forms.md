# Forms

Use shadcn/ui form primitives with tRPC mutations and Zod validation. Keep client interactivity in a `*Client.tsx` component and delegate data changes to mutations.

## When To Use

- Interactive input and validation
- Mutations that modify server state
- Accessible UI with consistent styling

## Basic Structure

```tsx
// src/components/features/tasks/TaskForm.tsx
'use client'

import { z } from 'zod'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { trpc, type RouterInputs } from '@/lib/api/react'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { useState, useCallback, type FormEvent } from 'react'

// Client-side validation schema (can differ from server schema)
const formSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100, 'Title too long'),
})

type FormData = z.infer<typeof formSchema>
type CreateTaskInput = RouterInputs['tasks']['create']

interface FormErrors {
  readonly title?: string
}

export function TaskForm() {
  const qc = useQueryClient()
  const [errors, setErrors] = useState<FormErrors>({})

  const create = useMutation({
    ...trpc.tasks.create.mutationOptions(),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: trpc.tasks.list.queryKey() })
    },
    onError: (error) => {
      console.error('Create failed:', error.message)
    },
  })

  const handleSubmit = useCallback(
    async (e: FormEvent<HTMLFormElement>) => {
      e.preventDefault()
      setErrors({})

      const fd = new FormData(e.currentTarget)
      const title = String(fd.get('title') ?? '')

      // Validate with Zod
      const result = formSchema.safeParse({ title })

      if (!result.success) {
        const fieldErrors: FormErrors = {}
        result.error.errors.forEach((err) => {
          if (err.path[0] === 'title') {
            fieldErrors.title = err.message
          }
        })
        setErrors(fieldErrors)
        return
      }

      // Type-safe mutation
      const input: CreateTaskInput = result.data
      await create.mutateAsync(input)
      e.currentTarget.reset()
    },
    [create]
  )

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <Input
          name="title"
          placeholder="New task"
          disabled={create.isPending}
          aria-invalid={!!errors.title}
          aria-describedby={errors.title ? 'title-error' : undefined}
        />
        {errors.title && (
          <p id="title-error" className="text-sm text-red-600 mt-1">
            {errors.title}
          </p>
        )}
      </div>
      <Button type="submit" disabled={create.isPending}>
        {create.isPending ? 'Adding...' : 'Add Task'}
      </Button>
    </form>
  )
}
```

## Complete Example

```ts
// src/lib/api/commands/mutations/tasks/create-task.mutation.ts
import { z } from 'zod'
import { db } from '@/lib/db'
import { tasks } from '@/lib/db/schema/tasks'

export const CreateTaskInputSchema = z.object({ title: z.string().min(1) }).strict()
export const CreateTaskOutputSchema = z.object({ id: z.number() })

export type CreateTaskInput = z.infer<typeof CreateTaskInputSchema>
export type CreateTaskOutput = z.infer<typeof CreateTaskOutputSchema>

export async function createTask(_ctx: { userId: string }, input: CreateTaskInput) {
  const [row] = await db.insert(tasks).values({ title: input.title }).returning({ id: tasks.id })
  return { id: row.id }
}
```

```ts
// src/lib/api/routers/tasks.router.ts
import { createTRPCRouter, rbacProcedure } from '../trpc'
import { CreateTaskInputSchema, CreateTaskOutputSchema } from '../commands/mutations/tasks/create-task.mutation'
import { createTask } from '../commands/mutations/tasks/create-task.mutation'

export const tasksRouter = createTRPCRouter({
  create: rbacProcedure.input(CreateTaskInputSchema).output(CreateTaskOutputSchema).mutation(({ ctx, input }) => createTask(ctx, input)),
})
```

## Common Mistakes

- Validating inside commands; validate at the router boundary
- Forgetting to invalidate or optimistically update related queries
- Overloading forms with too many responsibilities; split steps when needed

## Notes

- Localize validation messages with your i18n library; avoid raw user-visible strings

## How To

1. Define a client schema with Zod for form validation (client-only).
2. Wire a tRPC mutation with typed input/output.
3. In the client component, call `useMutation` and invalidate list queries on success.
4. Use shadcn/ui inputs and buttons for consistent accessibility and styling.

## Anti-pattern

- Validating inside commands/routers for client-only concerns; duplicate minimal client validation and rely on server validation for trust
- Mutations without cache invalidation or optimistic UX when appropriate

## Related

- mutations.md
- shadcn.md
- trpc-procedures.md
