# Server Components

## When to Use
- Default for pages in `app/`. Fetch data on the server.
- Render UI using data from tRPC via `createCaller` (no HTTP).
- Compose with small client components for interactivity.

## Structure
```
app/
  posts/
    page.tsx          # Server Component (async)
server/
  api/routers/
    posts.ts          # tRPC router
  trpc.ts             # init + createCaller helper
```

## Complete Example
File: `src/server/trpc.ts`
```ts
import { initTRPC } from '@trpc/server'
import { appRouter } from './api/routers/_app'

export const t = initTRPC.create()
export type AppRouter = typeof appRouter

export async function createContext() {
  return { /* user, db, etc. */ }
}

export async function createCaller() {
  return appRouter.createCaller(await createContext())
}
```

File: `src/app/posts/page.tsx`
```tsx
import { createCaller } from '@/server/trpc'

export default async function Page() {
  const caller = await createCaller()
  const posts = await caller.posts.list()
  return (
    <main className="space-y-4">
      <h1 className="text-2xl font-semibold">Posts</h1>
      <ul className="space-y-2">
        {posts.map((p) => (
          <li key={p.id}>{p.title}</li>
        ))}
      </ul>
    </main>
  )
}
```

## Anti-Patterns
❌ Fetch via HTTP to your own tRPC API in a Server Component.
✅ Use `createCaller` to avoid network overhead and keep types.

❌ Mix client hooks (`useState`, `useEffect`) in Server Components.
✅ Isolate interactivity into small `"use client"` components.

## Checklist
- [ ] Server component is `async`
- [ ] Uses `createCaller()` for data
- [ ] No client hooks in server files
- [ ] Child client components only for interaction

## Related Patterns
- See: `client-components.md`
- See: `../data/queries.md`
- See: `../routing/app-router.md`

