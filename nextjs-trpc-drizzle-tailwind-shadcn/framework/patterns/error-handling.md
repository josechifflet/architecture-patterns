# Error Handling

Use `TRPCError` and a shared formatter. Commands throw domain errors; routers map to tRPC errors.

## When To Use

- Protect procedures with RBAC
- Convert domain errors into safe client responses

## Setup

```ts
// src/lib/api/trpc.ts
import { initTRPC, TRPCError } from '@trpc/server'
import type { TRPC_ERROR_CODE_KEY } from '@trpc/server/rpc'

// Discriminated union for auth state
type UnauthenticatedContext = {
  userId: null
  permissions: null
}

type AuthenticatedContext = {
  userId: string
  permissions: readonly string[]
}

type Context = UnauthenticatedContext | AuthenticatedContext

const t = initTRPC.context<Context>().create({
  errorFormatter({ shape, error }) {
    // Redact internal messages in production
    const isProd = process.env.NODE_ENV === 'production'
    const isInternalError = error.code === 'INTERNAL_SERVER_ERROR'

    return {
      ...shape,
      message: isProd && isInternalError ? 'Internal server error' : shape.message,
    }
  },
})

export const createTRPCRouter = t.router
export const publicProcedure = t.procedure

// Type-safe RBAC middleware that narrows context
export const rbacProcedure = t.procedure.use(async ({ ctx, next }) => {
  if (!ctx.userId) {
    throw new TRPCError({
      code: 'UNAUTHORIZED',
      message: 'Authentication required',
    })
  }

  // Type assertion is safe here because we checked userId
  return next({
    ctx: ctx as AuthenticatedContext,
  })
})

// Exhaustive error codes with const assertion
const APP_ERROR_CODES = [
  'UNAUTHORIZED',
  'FORBIDDEN',
  'NOT_FOUND',
  'BAD_REQUEST',
  'CONFLICT',
  'UNPROCESSABLE_ENTITY',
  'INTERNAL',
] as const

type AppErrorCode = (typeof APP_ERROR_CODES)[number]

// Structured domain error with metadata
export interface AppError {
  readonly code: AppErrorCode
  readonly message: string
  readonly meta?: Readonly<Record<string, unknown>>
  readonly cause?: unknown
}

// Type guard with runtime validation
export function isAppError(e: unknown): e is AppError {
  if (typeof e !== 'object' || e === null) return false

  const err = e as Record<string, unknown>

  return (
    typeof err.code === 'string' &&
    APP_ERROR_CODES.includes(err.code as AppErrorCode) &&
    typeof err.message === 'string'
  )
}

// Exhaustive mapping with const assertion
const codeMap = {
  UNAUTHORIZED: 'UNAUTHORIZED',
  FORBIDDEN: 'FORBIDDEN',
  NOT_FOUND: 'NOT_FOUND',
  BAD_REQUEST: 'BAD_REQUEST',
  CONFLICT: 'CONFLICT',
  UNPROCESSABLE_ENTITY: 'UNPROCESSABLE_CONTENT',
  INTERNAL: 'INTERNAL_SERVER_ERROR',
} as const satisfies Record<AppErrorCode, TRPC_ERROR_CODE_KEY>

// Central error mapper with proper type narrowing
export function mapAppError(e: unknown): never {
  if (isAppError(e)) {
    throw new TRPCError({
      code: codeMap[e.code],
      message: e.message,
      cause: e.meta ?? e.cause,
    })
  }

  // Log unexpected errors for debugging
  console.error('Unexpected error:', e)

  // Re-throw as internal error for safety
  throw new TRPCError({
    code: 'INTERNAL_SERVER_ERROR',
    message: 'An unexpected error occurred',
    cause: e,
  })
}

// Helper to create typed domain errors
export function createAppError(
  code: AppErrorCode,
  message: string,
  meta?: Record<string, unknown>
): AppError {
  return { code, message, meta }
}
```

## Example

```ts
// Command layer - throw domain errors
// src/lib/api/commands/queries/tasks/get-task.query.ts
import { createAppError, type AppError } from '@/lib/api/trpc'
import { db } from '@/lib/db'
import { tasks } from '@/lib/db/schema/tasks'
import { eq } from 'drizzle-orm'

export async function getTask(ctx: { userId: string }, id: number) {
  const [task] = await db
    .select()
    .from(tasks)
    .where(eq(tasks.id, id))
    .limit(1)

  if (!task) {
    throw createAppError('NOT_FOUND', `Task ${id} not found`, { taskId: id })
  }

  return task
}
```

```ts
// Router layer - map domain errors to tRPC errors
// src/lib/api/routers/tasks.router.ts
import { createTRPCRouter, rbacProcedure, mapAppError } from '../trpc'
import { z } from 'zod'
import { getTask } from '../commands/queries/tasks/get-task.query'

export const tasksRouter = createTRPCRouter({
  byId: rbacProcedure
    .input(z.object({ id: z.number().int().positive() }).strict())
    .output(z.object({ id: z.number(), title: z.string(), done: z.boolean() }))
    .query(async ({ ctx, input }) => {
      try {
        return await getTask(ctx, input.id)
      } catch (e) {
        mapAppError(e)
      }
    }),
})
```

## Notes

- Keep commands free of tRPC imports; throw domain errors.
- Centralize mapping for consistent codes and messages.

## How To

1. Add a central `mapAppError` that converts domain `{ code, message }` to `TRPCError`.
2. Wrap router procedure bodies in try/catch and call the mapper in catch.
3. Keep commands throwing plain domain errors only; avoid TRPC imports in commands.
4. Optionally redact internal messages in `errorFormatter`.

## Anti-pattern

- Throwing `TRPCError` from deep inside commands
- Inconsistent error-to-code mapping per procedure
- Exposing raw DB or unexpected error messages to clients
