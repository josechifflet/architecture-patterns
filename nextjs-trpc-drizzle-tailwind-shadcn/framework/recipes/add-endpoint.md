# Add Endpoint

Add a new tRPC procedure with validation, DB calls via a command, and typed output.

## Prerequisites

- tRPC router setup and fetch adapter mounted
- Drizzle schema for the domain

## Steps

1. Define schemas and command

```ts
// src/lib/api/commands/queries/users/get-user.query.ts
import { z } from 'zod'
import { db } from '@/lib/db'
import { users } from '@/lib/db/schema/users'
import { eq } from 'drizzle-orm'

// Input schema with strict validation
export const GetUserInputSchema = z
  .object({
    id: z.number().int().positive(),
  })
  .strict()

// Output schema matching selected fields
export const GetUserOutputSchema = z
  .object({
    id: z.number().int().positive(),
    name: z.string(),
    email: z.string().email(),
  })
  .strict()

// Export inferred types
export type GetUserInput = z.infer<typeof GetUserInputSchema>
export type GetUserOutput = z.infer<typeof GetUserOutputSchema>

// Type-safe context
type QueryContext = {
  readonly userId: string
  readonly permissions: readonly string[]
}

export async function getUser(
  _ctx: QueryContext,
  input: GetUserInput
): Promise<GetUserOutput | null> {
  const [row] = await db
    .select({
      id: users.id,
      name: users.name,
      email: users.email,
    })
    .from(users)
    .where(eq(users.id, input.id))
    .limit(1)

  return row ?? null
}
```

2. Wire the router

```ts
// src/lib/api/routers/users.router.ts
import { createTRPCRouter, rbacProcedure, createAppError, mapAppError } from '../trpc'
import {
  GetUserInputSchema,
  GetUserOutputSchema,
  getUser,
  type GetUserOutput,
} from '../commands/queries/users/get-user.query'

export const usersRouter = createTRPCRouter({
  byId: rbacProcedure
    .input(GetUserInputSchema.strict())
    .output(GetUserOutputSchema.strict())
    .query(async ({ ctx, input }) => {
      try {
        const user = await getUser(ctx, input)

        if (!user) {
          throw createAppError('NOT_FOUND', `User ${input.id} not found`, {
            userId: input.id,
          })
        }

        return user
      } catch (e) {
        mapAppError(e)
      }
    }),
})

// Export router type for client usage
export type UsersRouter = typeof usersRouter
```

3. Use from a client component

```tsx
// src/components/features/users/UserClient.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { trpc, type RouterOutputs } from '@/lib/api/react'
import { useTranslations } from 'next-intl'

type User = RouterOutputs['users']['byId']

interface UserClientProps {
  readonly id: number
}

export default function UserClient({ id }: UserClientProps) {
  const t = useTranslations('Users')

  const q = useQuery(trpc.users.byId.queryOptions({ id }))

  if (q.isPending) {
    return <div>{t('loading')}</div>
  }

  if (q.error) {
    return <div>{t('loadError', { error: q.error.message })}</div>
  }

  // q.data is fully typed as User
  return (
    <div>
      <h2>{q.data.name}</h2>
      <p>{q.data.email}</p>
    </div>
  )
}
```

## Verification

- Type errors surface on invalid input or output
- Not found errors are mapped by the router error mapper
- Query caches are invalidated when related mutations run
- Localized strings render for all supported locales

## Related

- patterns/trpc-procedures.md
- patterns/queries.md
