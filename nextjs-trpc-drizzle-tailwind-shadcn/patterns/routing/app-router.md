# App Router

## When to Use
- All routes in `src/app/**`. Prefer Server Components.

## Structure
```
app/
  layout.tsx
  page.tsx
  posts/
    page.tsx
  posts/[id]/
    page.tsx
  api/
    trpc/[trpc]/route.ts
```

## Complete Example
Dynamic segment:
```tsx
// app/posts/[id]/page.tsx
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  return <div>Post {id}</div>
}
```

Programmatic navigation (client):
```tsx
"use client"
import { useRouter } from 'next/navigation'

export function GoToDashboard() {
  const router = useRouter()
  return <button onClick={() => router.push('/dashboard')}>Go</button>
}
```

## Anti-Patterns
❌ Mixing Pages Router and App Router for new work.
✅ Use App Router exclusively.

## Checklist
- [ ] File-system driven routing only
- [ ] Folders = route segments
- [ ] `route.ts` only for handlers

## Related Patterns
- See: `route-handlers.md`
- See: `../components/layout-patterns.md`

