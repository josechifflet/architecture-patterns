# Server Components

Server Components render by default in the App Router and keep data fetching on the server. Use them for pages, layouts, and non‑interactive UI that can be streamed and cached. For server reads, prefer the tRPC server caller instead of importing DB/commands.

## When To Use

- Rendering route pages and layouts
- Composing data from multiple sources
- Passing data to client components for interactivity
- Streaming with `loading.tsx` and Suspense

## Basic Structure

```tsx
// app/(app)/dashboard/page.tsx
import Chart from '@/components/features/analytics/ChartClient'

export default async function Page() {
  // Orchestration only; fetch inside containers or use a server caller here
  return <Chart />
}
```

## Complete Example

```tsx
// app/(app)/tasks/page.tsx
import TasksClient from '@/components/features/tasks/TasksClient'

export default async function Page() {
  return <TasksClient />
}
```

```tsx
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

For route params and query strings in Server Components, `params` and `searchParams` are Promises in Next.js 15+. Always await them with proper typing.

```tsx
// app/(app)/tasks/[id]/page.tsx
import type { Metadata } from 'next'

// Type-safe params
interface PageParams {
  readonly id: string
}

interface PageProps {
  readonly params: Promise<PageParams>
  readonly searchParams?: Promise<Record<string, string | string[] | undefined>>
}

// Type-safe metadata generation
export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { id } = await params
  return {
    title: `Task ${id}`,
  }
}

export default async function Page({ params, searchParams }: PageProps) {
  const { id } = await params
  const search = await searchParams

  // id is typed as string, not string | string[]
  return (
    <div>
      <h1>Task {id}</h1>
      {search?.filter && <p>Filter: {search.filter}</p>}
    </div>
  )
}
```

Fetch via the server caller when reading in RSC with full type safety.

```tsx
// app/(app)/tasks/page.tsx
import { api } from '@/lib/api/server'
import type { RouterOutputs } from '@/lib/api/react'

type TasksList = RouterOutputs['tasks']['list']

export default async function Page() {
  const caller = await api()

  // data is fully typed as TasksList
  const data: TasksList = await caller.tasks.list({
    page: 1,
    pageSize: 10,
  })

  return (
    <div>
      <h2>Tasks ({data.totalCount})</h2>
      <ul>
        {data.items.map((task) => (
          <li key={task.id}>{task.title}</li>
        ))}
      </ul>
      <p>
        Page {data.page} of {data.totalPages}
      </p>
    </div>
  )
}
```

## Caching & Revalidation

- Make cache intent explicit on `fetch` when used in RSC
- Use `cache: 'no-store'` for highly dynamic user data, or `next: { revalidate: n }` for ISR
- Tag `fetch` requests with `next: { tags: [...] }` to revalidate on demand

For prefetch + hydrate patterns with tRPC v11 and TanStack Query v5, see rsc-prefetch-hydration.md.
```

## Common Mistakes

- Importing DB or commands directly in UI. Use the API server caller or client hooks
- Forgetting to await `params` or `searchParams` in Next.js 15
- Doing heavy data work in client components instead of on the server

## Related

- client-components.md
- routes.md
- queries.md
- providers.md

## How To

1. Keep pages/layouts as Server Components by default.
2. For server reads, use a server caller or fetch with explicit cache intent.
3. Pass minimal props to client islands that need interactivity.
4. Await `params`/`searchParams` in Next 15+ Server Components.

## Anti-pattern

- Importing DB/schema/commands in UI components
- Doing client-side data joins/aggregation that belong on the server
- Not declaring caching intent on server fetches when needed
