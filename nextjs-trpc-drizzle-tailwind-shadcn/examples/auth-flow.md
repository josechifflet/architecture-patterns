# Example: Auth Flow (Session Cookie + tRPC Middleware)

## Overview
Minimal auth: read a signed cookie, load user in context, guard procedures with middleware.

## Files
```
server/auth.ts
server/trpc.ts
server/api/routers/_app.ts
server/api/routers/secure.ts
app/login/page.tsx
```

## Context + Middleware
```ts
// server/auth.ts
import { cookies } from 'next/headers'
import { db } from '@/server/db'
import { users } from '@/server/db/schema/users'

export async function getSessionUser() {
  const id = (await cookies()).get('sid')?.value
  if (!id) return null
  const [u] = await db.select().from(users).where(users.id.eq(Number(id)))
  return u ?? null
}

// server/trpc.ts
import { initTRPC } from '@trpc/server'
import { getSessionUser } from './auth'

export const t = initTRPC.context<awaited<ReturnType<typeof createContext>>>().create()

export async function createContext() {
  const user = await getSessionUser()
  return { user }
}

const isAuthed = t.middleware(({ ctx, next }) => {
  if (!ctx.user) throw new Error('UNAUTHORIZED')
  return next({ ctx })
})

export const protectedProcedure = t.procedure.use(isAuthed)
```

## Secure Router
```ts
// server/api/routers/secure.ts
import { protectedProcedure, t } from '@/server/trpc'

export const secureRouter = t.router({
  me: protectedProcedure.query(({ ctx }) => ctx.user),
})
```

## Login Page (Example)
```tsx
// app/login/page.tsx
export default function Page() {
  async function login(formData: FormData) {
    'use server'
    // verify credentials then set a cookie via headers()
  }
  return (
    <form action={login} className="space-y-3">
      <input name="email" className="border p-2" />
      <input name="password" type="password" className="border p-2" />
      <button className="px-3 py-1 rounded bg-zinc-900 text-white">Login</button>
    </form>
  )
}
```

## Notes
- Replace cookie logic with your identity provider as needed.
- Consider rotating session IDs and using `Secure; HttpOnly; SameSite` flags.

