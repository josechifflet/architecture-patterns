# tRPC Procedures

## When to Use
- Any server boundary for data access or mutations
- Validate inputs at the edge of your server logic

## Structure
```
server/
  api/
    routers/
      posts.ts          # feature router
      _app.ts           # merge routers
  trpc.ts               # tRPC init, context, createCaller
app/
  api/trpc/[trpc]/route.ts   # fetchRequestHandler endpoint
```

## Complete Example
File: `src/server/api/routers/posts.ts`
```ts
import { z } from 'zod'
import { t } from '@/server/trpc'
import { db } from '@/server/db'
import { posts } from '@/server/db/schema/posts'

export const postsRouter = t.router({
  list: t.procedure
    .query(async () => {
      return await db.select().from(posts).orderBy(posts.created_at.desc())
    }),

  create: t.procedure
    .input(z.object({ title: z.string().min(3) }))
    .mutation(async ({ input }) => {
      const [row] = await db.insert(posts).values({ title: input.title }).returning()
      return row
    }),
})
```

File: `src/server/api/routers/_app.ts`
```ts
import { t } from '@/server/trpc'
import { postsRouter } from './posts'

export const appRouter = t.router({
  posts: postsRouter,
})
```

File: `src/app/api/trpc/[trpc]/route.ts`
```ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@/server/api/routers/_app'
import { createContext } from '@/server/trpc'

const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: '/api/trpc',
    req,
    router: appRouter,
    createContext,
  })

export { handler as GET, handler as POST }
```

## Anti-Patterns
❌ Mixing side-effects and queries in one procedure.
✅ Separate `list/get` queries from `create/update/delete` mutations.

## Checklist
- [ ] Zod-validated inputs
- [ ] No direct DB access from UI — always via procedures
- [ ] All procedures names are `verbNoun`
- [ ] Feature routers merged in `_app.ts`

## Related Patterns
- See: `mutations.md`, `queries.md`
- See: `../routing/route-handlers.md`

