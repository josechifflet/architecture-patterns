# Auth Flow

Protect routes and procedures with a server context that resolves the current user and applies RBAC. Use App Router for UI guards and tRPC middleware for server protection.

## Context

- End‑to‑end type safety
- Server‑first auth checks
- UI gated by server‑derived permissions

## Server Context

```ts
// src/lib/api/context.ts
import { cookies } from 'next/headers'

export async function createContext() {
  const jar = await cookies()
  const session = jar.get('session')?.value
  const user = session ? await verifySession(session) : null
  return { userId: user?.id }
}

async function verifySession(token: string) {
  // verify and return user or null
  return { id: 'user_123' }
}
```

## RBAC Middleware

```ts
// src/lib/api/trpc.ts
import { initTRPC } from '@trpc/server'
import { TRPCError } from '@trpc/server'

const t = initTRPC.context<{ userId?: string }>().create()

export const createTRPCRouter = t.router
export const publicProcedure = t.procedure
export const rbacProcedure = t.procedure.use(async ({ ctx, next }) => {
  if (!ctx.userId) throw new TRPCError({ code: 'UNAUTHORIZED' })
  return next()
})
```

## Protected Procedure

```ts
// src/lib/api/routers/profile.router.ts
import { createTRPCRouter, rbacProcedure } from '../trpc'
import { z } from 'zod'

export const profileRouter = createTRPCRouter({
  me: rbacProcedure.output(z.object({ id: z.string() })).query(({ ctx }) => ({ id: String(ctx.userId) })),
})
```

## UI Guard

```tsx
// app/(app)/settings/page.tsx
import SettingsClient from '@/components/features/settings/SettingsClient'

export default async function Page() {
  return <SettingsClient />
}
```

```tsx
// src/components/features/settings/SettingsClient.tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { trpc } from '@/lib/api/react'

export default function SettingsClient() {
  const me = useQuery(trpc.profile.me.queryOptions())
  if (me.isPending) return <div>Loading…</div>
  if (me.error) return <div>Sign in required</div>
  return <div>User: {me.data.id}</div>
}
```

## Verification

- Unauthenticated requests receive an error from protected procedures
- UI renders a signed‑out state when `me` fails
- Signed‑in users see settings content

## Related

- ../patterns/trpc-procedures.md
- ../patterns/server-components.md
