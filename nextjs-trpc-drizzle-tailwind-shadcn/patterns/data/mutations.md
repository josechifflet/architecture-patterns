# Mutations

## When to Use
- Create, update, delete operations via tRPC procedures
- Server authority for business rules and validation

## Structure
```
server/api/routers/
  posts.ts        # define `create`, `update`, `delete`
```

## Complete Example
File: `src/server/api/routers/posts.ts` (excerpt)
```ts
import { z } from 'zod'
import { t } from '@/server/trpc'
import { db } from '@/server/db'
import { posts } from '@/server/db/schema/posts'

export const postsRouter = t.router({
  create: t.procedure
    .input(z.object({ title: z.string().min(3) }))
    .mutation(async ({ input }) => {
      const [row] = await db.insert(posts).values({ title: input.title }).returning()
      return row
    }),

  update: t.procedure
    .input(z.object({ id: z.number().int().positive(), title: z.string().min(3) }))
    .mutation(async ({ input }) => {
      const [row] = await db
        .update(posts)
        .set({ title: input.title })
        .where(posts.id.eq(input.id))
        .returning()
      return row
    }),

  remove: t.procedure
    .input(z.object({ id: z.number().int().positive() }))
    .mutation(async ({ input }) => {
      await db.delete(posts).where(posts.id.eq(input.id))
      return { ok: true }
    }),
})
```

## Anti-Patterns
❌ Returning raw DB errors to clients.
✅ Normalize and surface friendly messages, log details server-side.

❌ Side-effects without idempotency.
✅ Make mutations safe to retry (e.g., unique keys).

## Checklist
- [ ] Zod validation at boundary
- [ ] Idempotent or safe retries where possible
- [ ] Clear return shape for UI consumption

## Related Patterns
- See: `../components/form-patterns.md`
- See: `trpc-procedures.md`

