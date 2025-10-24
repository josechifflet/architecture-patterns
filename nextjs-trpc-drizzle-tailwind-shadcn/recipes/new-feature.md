# Recipe: Add New Feature

## Prerequisites
- [ ] Next.js 16.0.0
- [ ] tRPC 11.6.0
- [ ] Drizzle ORM 0.44.7
- [ ] Tailwind 4.1.16 + PostCSS plugin
- [ ] shadcn CLI 3.5.0

## Steps

### 1. Database Schema
File: `src/server/db/schema/[feature].ts`
```
import { pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core'

export const items = pgTable('items', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  created_at: timestamp('created_at').defaultNow().notNull(),
})
```

Generate + apply migrations:
```
npx drizzle-kit generate
npx drizzle-kit migrate
```

### 2. tRPC Router
File: `src/server/api/routers/[feature].ts`
```
import { z } from 'zod'
import { t } from '@/server/trpc'
import { db } from '@/server/db'
import { items } from '@/server/db/schema/items'

export const itemsRouter = t.router({
  list: t.procedure.query(() => db.select().from(items)),
  create: t.procedure
    .input(z.object({ name: z.string().min(1) }))
    .mutation(async ({ input }) => {
      const [row] = await db.insert(items).values({ name: input.name }).returning()
      return row
    }),
})
```

Add to app router: `src/server/api/routers/_app.ts`
```
import { t } from '@/server/trpc'
import { itemsRouter } from './items'

export const appRouter = t.router({
  items: itemsRouter,
})
```

### 3. Server Page
File: `src/app/items/page.tsx`
```
import { createCaller } from '@/server/trpc'
import { ItemCreator } from '@/components/ItemCreator'

export default async function Page() {
  const caller = await createCaller()
  const items = await caller.items.list()
  return (
    <section className="space-y-4">
      <h1 className="text-2xl font-semibold">Items</h1>
      <ItemCreator />
      <ul className="space-y-2">
        {items.map(i => <li key={i.id}>{i.name}</li>)}
      </ul>
    </section>
  )
}
```

### 4. Client Component
File: `src/components/ItemCreator.tsx`
```
"use client"
import { useState } from 'react'
import { trpc } from '@/lib/trpcClient'
import { Button, Input } from '@/components/ui'

export function ItemCreator() {
  const [name, setName] = useState('')
  async function onCreate() {
    await trpc.items.create.mutate({ name })
    setName('')
  }
  return (
    <div className="flex gap-2">
      <Input value={name} onChange={e => setName(e.target.value)} placeholder="Name" />
      <Button onClick={onCreate}>Add</Button>
    </div>
  )
}
```

## Verification
- [ ] Types compile and infer correctly
- [ ] `GET /app/items` renders list
- [ ] Mutations work from the UI

## Troubleshooting
Problem: 404 on `/api/trpc`.
Solution: Ensure `app/api/trpc/[trpc]/route.ts` exists and exports `GET` and `POST` using `fetchRequestHandler`.

