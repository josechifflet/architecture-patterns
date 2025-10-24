# Queries

## When to Use
- Read-only data access from DB via tRPC
- Prefer server-side fetching in Server Components

## Structure
```
server/api/routers/
  posts.ts        # define `list`, `getById`
server/trpc.ts    # `createCaller()` for server-side usage
```

## Complete Example
Server-side (RSC) via `createCaller`:
```tsx
import { createCaller } from '@/server/trpc'

export default async function Page() {
  const caller = await createCaller()
  const posts = await caller.posts.list()
  return <pre>{JSON.stringify(posts, null, 2)}</pre>
}
```

Client-side via vanilla tRPC client:
```tsx
"use client"
import { useEffect, useState } from 'react'
import { trpc } from '@/lib/trpcClient'

export function PostsClientList() {
  const [data, setData] = useState<any[]>([])
  useEffect(() => { trpc.posts.list.query().then(setData) }, [])
  return <ul>{data.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

## Anti-Patterns
❌ N+1 queries inside procedures.
✅ Use proper joins or batched queries.

❌ Fetching on client when RSC suffices.
✅ Prefer server fetching for initial render.

## Checklist
- [ ] Use RSC for initial data
- [ ] tRPC procedures are read-only
- [ ] Stable return shapes for UI

## Related Patterns
- See: `server-components.md`
- See: `trpc-procedures.md`

