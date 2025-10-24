# Client Components

## When to Use
- Interactivity: forms, buttons, navigation, real-time states
- Call tRPC from the browser for mutations or user-scoped queries

## Structure
```
lib/
  trpcClient.ts        # tiny tRPC vanilla client
components/
  PostCreator.tsx      # "use client" interactive component
```

## Complete Example
File: `src/lib/trpcClient.ts`
```ts
import { createTRPCClient, httpBatchLink } from '@trpc/client'
import type { AppRouter } from '@/server/trpc'

export const trpc = createTRPCClient<AppRouter>({
  links: [
    httpBatchLink({ url: '/api/trpc' }),
  ],
})
```

File: `src/components/PostCreator.tsx`
```tsx
"use client"
import { useState } from 'react'
import { trpc } from '@/lib/trpcClient'
import { Button, Input } from '@/components/ui'

export function PostCreator() {
  const [title, setTitle] = useState('')

  async function handleCreate() {
    await trpc.posts.create.mutate({ title })
    setTitle('')
    // Optionally refetch page via router.refresh()
  }

  return (
    <div className="flex items-center gap-2">
      <Input value={title} onChange={e => setTitle(e.target.value)} placeholder="Title" />
      <Button onClick={handleCreate}>Create</Button>
    </div>
  )
}
```

## Anti-Patterns
❌ Fetch on every keystroke without debounce or intent.
✅ Trigger mutations on explicit user actions.

❌ Put secrets or DB logic in client code.
✅ All server logic goes through tRPC procedures.

## Checklist
- [ ] `"use client"` at top of file
- [ ] Calls tRPC client for data changes
- [ ] No secrets, keys, or DB imports

## Related Patterns
- See: `form-patterns.md`
- See: `../data/mutations.md`
- See: `../routing/route-handlers.md`

