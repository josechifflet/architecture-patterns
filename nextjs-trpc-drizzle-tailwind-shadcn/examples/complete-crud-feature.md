# Example: Complete CRUD Feature (Posts)

## Overview
End-to-end: schema → router → pages/components → route handler.

## Files
```
server/db/schema/posts.ts
server/api/routers/posts.ts
server/api/routers/_app.ts
app/api/trpc/[trpc]/route.ts
app/posts/page.tsx
components/PostCreator.tsx
```

## Schema
```ts
import { pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core'
export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  created_at: timestamp('created_at').defaultNow().notNull(),
})
```

## Router
```ts
import { z } from 'zod'
import { t } from '@/server/trpc'
import { db } from '@/server/db'
import { posts } from '@/server/db/schema/posts'

export const postsRouter = t.router({
  list: t.procedure.query(() => db.select().from(posts)),
  create: t.procedure.input(z.object({ title: z.string().min(3) }))
    .mutation(async ({ input }) => (await db.insert(posts).values({ title: input.title }).returning())[0]),
  remove: t.procedure.input(z.object({ id: z.number().int().positive() }))
    .mutation(async ({ input }) => { await db.delete(posts).where(posts.id.eq(input.id)); return { ok: true } }),
})
```

## App Router Merge
```ts
export const appRouter = t.router({ posts: postsRouter })
```

## tRPC Endpoint
```ts
const handler = (req: Request) => fetchRequestHandler({ endpoint: '/api/trpc', req, router: appRouter, createContext })
export { handler as GET, handler as POST }
```

## Server Page
```tsx
import { createCaller } from '@/server/trpc'
import { PostCreator } from '@/components/PostCreator'

export default async function Page() {
  const caller = await createCaller()
  const posts = await caller.posts.list()
  return (
    <section className="space-y-4">
      <h1 className="text-2xl font-semibold">Posts</h1>
      <PostCreator />
      <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
    </section>
  )
}
```

## Client Component
```tsx
"use client"
import { trpc } from '@/lib/trpcClient'
import { useState } from 'react'
import { Button, Input } from '@/components/ui'

export function PostCreator() {
  const [title, setTitle] = useState('')
  return (
    <div className="flex gap-2">
      <Input value={title} onChange={(e) => setTitle(e.target.value)} />
      <Button onClick={async () => { await trpc.posts.create.mutate({ title }); setTitle('') }}>Create</Button>
    </div>
  )
}
```

## Verification
- [ ] CRUD works
- [ ] Page loads with server-fetched list
- [ ] Mutation updates persist after refresh

