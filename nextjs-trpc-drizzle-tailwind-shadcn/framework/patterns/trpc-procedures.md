# tRPC Procedures

Define thin routers with input/output schemas and delegate to commands. Use v11 with TanStack Query v5 for client hooks and utilities. Map domain errors centrally; do not inline TRPC errors.

## When To Use

- Server API contracts for queries and mutations
- Input validation with Zod
- RBAC and middleware at the procedure level

## Basic Structure

```ts
// src/lib/api/trpc.ts
import { initTRPC, type inferAsyncReturnType } from '@trpc/server'
import { TRPCError } from '@trpc/server'
import type { TRPC_ERROR_CODE_KEY } from '@trpc/server/rpc'

// Type-safe context with discriminated union for auth state
type UnauthenticatedContext = {
  userId: null
  permissions: null
}

type AuthenticatedContext = {
  userId: string
  permissions: readonly string[]
}

export type Context = UnauthenticatedContext | AuthenticatedContext

const t = initTRPC.context<Context>().create()

export const createTRPCRouter = t.router
export const publicProcedure = t.procedure
export const rbacProcedure = t.procedure // attach permission checks in middleware

// Central error handling with branded types for exhaustiveness
const APP_ERROR_CODES = ['UNAUTHORIZED', 'FORBIDDEN', 'NOT_FOUND', 'BAD_REQUEST', 'INTERNAL'] as const
type AppErrorCode = (typeof APP_ERROR_CODES)[number]

interface AppError {
  readonly code: AppErrorCode
  readonly message?: string
  readonly meta?: Record<string, unknown>
}

export function isAppError(e: unknown): e is AppError {
  return (
    typeof e === 'object' &&
    e !== null &&
    'code' in e &&
    typeof (e as { code: unknown }).code === 'string' &&
    APP_ERROR_CODES.includes((e as { code: string }).code as AppErrorCode)
  )
}

const codeMap = {
  UNAUTHORIZED: 'UNAUTHORIZED',
  FORBIDDEN: 'FORBIDDEN',
  NOT_FOUND: 'NOT_FOUND',
  BAD_REQUEST: 'BAD_REQUEST',
  INTERNAL: 'INTERNAL_SERVER_ERROR',
} as const satisfies Record<AppErrorCode, TRPC_ERROR_CODE_KEY>

export function mapAppErrorToTRPC(e: AppError): never {
  throw new TRPCError({
    code: codeMap[e.code],
    message: e.message,
    cause: e.meta,
  })
}
```

## Complete Example

```ts
// src/lib/api/routers/tasks.router.ts
import { z } from 'zod'
import { createTRPCRouter, rbacProcedure, type Context } from '../trpc'
import type { inferProcedureInput, inferProcedureOutput } from '@trpc/server'
import {
  ListTasksInputSchema,
  ListTasksOutputSchema,
  listTasks,
} from '../commands/queries/tasks/list-tasks.query'
import {
  ToggleTaskInputSchema,
  ToggleTaskOutputSchema,
  toggleTask,
} from '../commands/mutations/tasks/toggle-task.mutation'

export const tasksRouter = createTRPCRouter({
  list: rbacProcedure
    .input(ListTasksInputSchema.strict())
    .output(ListTasksOutputSchema.strict())
    .query(async ({ ctx, input }) => {
      // ctx is narrowed to AuthenticatedContext by rbacProcedure
      return listTasks(ctx, input)
    }),

  toggle: rbacProcedure
    .input(ToggleTaskInputSchema.strict())
    .output(ToggleTaskOutputSchema.strict())
    .mutation(async ({ ctx, input }) => {
      return toggleTask(ctx, input)
    }),
})

// Export inferred types for type-safe client usage
export type TasksRouter = typeof tasksRouter
export type ListTasksInput = inferProcedureInput<TasksRouter['list']>
export type ListTasksOutput = inferProcedureOutput<TasksRouter['list']>
```

Expose the router using the Next.js fetch adapter in a route handler.

```ts
// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@/lib/api/routers/_app'
import { createContext } from '@/lib/api/context'

const handler = (req: Request) =>
  fetchRequestHandler({ endpoint: '/api/trpc', req, router: appRouter, createContext })

export { handler as GET, handler as POST }
```

Client integration uses TanStack Query v5 with full type inference.

```ts
// src/lib/api/react.ts
import { createTRPCReact } from '@trpc/react-query'
import type { AppRouter } from '@/lib/api/routers/_app'
import type { inferRouterInputs, inferRouterOutputs } from '@trpc/server'

export const trpc = createTRPCReact<AppRouter>()

// Export type helpers for client usage
export type RouterInputs = inferRouterInputs<AppRouter>
export type RouterOutputs = inferRouterOutputs<AppRouter>
```

```tsx
// usage in a Client Component with full type safety
'use client'

import { useQuery } from '@tanstack/react-query'
import { trpc, type RouterOutputs } from '@/lib/api/react'

type TasksList = RouterOutputs['tasks']['list']

export function TasksClient() {
  const list = useQuery(
    trpc.tasks.list.queryOptions({
      page: 1,
      pageSize: 20,
    })
  )

  if (list.isPending) return <div>Loading...</div>
  if (list.error) return <div>Error: {list.error.message}</div>

  // list.data is fully typed as TasksList
  return <div>Total: {list.data.totalCount}</div>
}

// Or use the RSC server caller in Server Components (preferred for RSC)
// import { api } from '@/lib/api/server'
// const caller = await api()
// const result = await caller.tasks.list({ page: 1, pageSize: 20 })
```

## Common Mistakes

- Validating inside commands instead of at the router boundary
- Inlining TRPC errors; use a centralized error mapper
- Mixing RBAC and business logic; keep RBAC in middleware

## Notes

- Prefer `rbacProcedure` for protected routes; reserve simpler guards for auth-only checks
- Server Components read via a server caller (`const caller = await api()`); never import DB/commands in RSC/UI
- Map unauthorized to UNAUTHORIZED and permission-denied to FORBIDDEN via the shared error mapper

## Related

- queries.md
- mutations.md
- server-components.md

## How To

1. Define Zod input/output schemas colocated with the command in `commands/queries|mutations`.
2. Implement the command function that receives `(ctx, input)` and performs DB work.
3. Wire the router with `.input().output().query|mutation()` delegating to the command.
4. Map errors via a central mapper; do not throw `TRPCError` in commands.

## Anti-pattern

- Validating inside commands instead of at the router boundary
- Inlining TRPC errors in each procedure
- Mixing RBAC and business logic inside the same handler
