# RSC Prefetch & Hydration

Prefetch queries on the server, hydrate on the client for instant, type-safe data using tRPC v11 + TanStack Query v5.

## When To Use

- SEO-friendly pages that benefit from server data
- Reduce client waterfalls and double-fetching
- Keep interactivity limited to islands

## Setup

```ts
// src/lib/api/server-trpc.ts (server-only)
import 'server-only'
import { cache } from 'react'
import { createTRPCOptionsProxy } from '@trpc/tanstack-react-query'
import { appRouter } from '@/lib/api/routers/_app'
import { createContext } from '@/lib/api/context'
import { makeQueryClient } from '@/lib/api/query-client'

export const getQueryClient = cache(makeQueryClient)

export const trpcServer = createTRPCOptionsProxy({
  ctx: createContext,
  router: appRouter,
  queryClient: getQueryClient,
})
```

## Page Example

```tsx
// app/(app)/tasks/page.tsx
import { dehydrate, HydrationBoundary } from '@tanstack/react-query'
import { getQueryClient, trpcServer } from '@/lib/api/server-trpc'
import TasksClient from '@/components/features/tasks/TasksClient'

export default async function Page() {
  const qc = getQueryClient()
  await qc.prefetchQuery(
    trpcServer.tasks.list.queryOptions({ page: 1, pageSize: 20 })
  )
  return (
    <HydrationBoundary state={dehydrate(qc)}>
      <TasksClient />
    </HydrationBoundary>
  )
}
```

## Client Example

```tsx
'use client'
import { useQuery } from '@tanstack/react-query'
import { trpc } from '@/lib/api/react'

export default function TasksClient() {
  const list = useQuery(trpc.tasks.list.queryOptions({ page: 1, pageSize: 20 }))
  if (list.isPending) return <div>Loading…</div>
  return <div>Total: {list.data.totalCount}</div>
}
```

## Notes

- Use this pattern when you need hydrated client interactivity; otherwise, render data directly in RSC with the server caller.
- Stick to RSC for heavy joins/reads; hydrate the minimal interactive shells.

## How To

1. Create a server-side TRPC options proxy and a cached QueryClient factory.
2. In the page (RSC), `prefetchQuery` using `trpcServer.*.queryOptions()`.
3. Wrap children in `<HydrationBoundary state={dehydrate(qc)}>`.
4. In the client island, call `useQuery` with the same `queryOptions`.

## Anti-pattern

- Double-fetching by calling both server caller and client `useQuery` for the same data without hydration
- Hydrating large trees unnecessarily; limit to the interactive subtree
