# Client Components

Client Components opt into interactivity with the `use client` directive. Use them for state, effects, event handlers, and navigation hooks.

## When To Use

- Inputs, forms, and interactive controls
- Hooks like `useState`, `useEffect`, or `useRouter`
- Rendering data fetched via React Query

## Basic Structure

```tsx
// src/components/features/tasks/TasksClient.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { trpc, type RouterOutputs } from '@/lib/api/react'

// Derive types from router outputs
type TasksList = RouterOutputs['tasks']['list']
type Task = TasksList['items'][number]

interface TasksClientProps {
  readonly initialPage?: number
  readonly initialPageSize?: number
}

export default function TasksClient({
  initialPage = 1,
  initialPageSize = 20,
}: TasksClientProps) {
  const list = useQuery(
    trpc.tasks.list.queryOptions({
      page: initialPage,
      pageSize: initialPageSize,
    })
  )

  // Exhaustive loading/error/success states
  if (list.isPending) return <div>Loading…</div>
  if (list.error) return <div>Error: {list.error.message}</div>

  // list.data is fully typed as TasksList
  return (
    <ul>
      {list.data.items.map((task: Task) => (
        <li key={task.id}>{task.title}</li>
      ))}
    </ul>
  )
}
```

## Complete Example

```tsx
// src/components/features/tasks/TaskActionsClient.tsx
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'
import { trpc, type RouterOutputs, type RouterInputs } from '@/lib/api/react'
import { Button } from '@/components/ui/button'
import { useCallback } from 'react'

// Derive types from router
type ToggleTaskInput = RouterInputs['tasks']['toggle']
type ToggleTaskOutput = RouterOutputs['tasks']['toggle']

interface TaskActionsClientProps {
  readonly id: number
  readonly onToggleSuccess?: (data: ToggleTaskOutput) => void
}

export function TaskActionsClient({
  id,
  onToggleSuccess,
}: TaskActionsClientProps) {
  const qc = useQueryClient()

  const toggle = useMutation({
    ...trpc.tasks.toggle.mutationOptions(),
    onSuccess: (data) => {
      // Invalidate related queries
      qc.invalidateQueries({ queryKey: trpc.tasks.list.queryKey() })

      // Call optional success handler
      onToggleSuccess?.(data)
    },
    onError: (error) => {
      console.error('Toggle failed:', error.message)
    },
  })

  const handleToggle = useCallback(() => {
    const input: ToggleTaskInput = { id }
    toggle.mutate(input)
  }, [id, toggle])

  return (
    <Button
      onClick={handleToggle}
      disabled={toggle.isPending}
      variant="secondary"
    >
      {toggle.isPending ? 'Toggling...' : 'Toggle'}
    </Button>
  )
}
```

## Common Mistakes

- Missing `use client` directive at the top of the file
- Doing data fetching that belongs in a server boundary
- Coupling to router state without `next/navigation` hooks

## How To

1. Create a `*Client.tsx` component and put `"use client"` at the top.
2. Use tRPC v11 + TanStack Query v5 via `trpc.*.queryOptions()` and `useQuery`.
3. Render `isPending`/error states and keep the component focused on interactivity.
4. Invalidate queries after mutations via `queryClient.invalidateQueries({ queryKey })`.

## Anti-pattern

- Fetching heavy data in client when RSC can render it server-side
- Importing server commands or DB from UI
- Omitting loading/error states for async client data

## Related

- server-components.md
- trpc-procedures.md
- forms.md
