# Recipe: Add New Page

## Prerequisites
- [ ] Layout exists (`app/layout.tsx`)
- [ ] `globals.css` imports Tailwind v4

## Steps

### 1. Create Route Folder
File: `src/app/[route]/page.tsx`
```
export default function Page() {
  return <h1 className="text-2xl font-semibold">Hello</h1>
}
```

### 2. Optional Dynamic Route
File: `src/app/users/[id]/page.tsx`
```
export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  return <div>User {id}</div>
}
```

### 3. Add Client Interactivity (Optional)
File: `src/components/GoButton.tsx`
```
"use client"
import { useRouter } from 'next/navigation'
export function GoButton() {
  const router = useRouter()
  return <button onClick={() => router.push('/dashboard')}>Go</button>
}
```

## Verification
- [ ] Page renders at new route
- [ ] Layout wraps content
- [ ] Client interactions work (if added)

## Troubleshooting
Problem: Styling missing.
Solution: Ensure `app/layout.tsx` imports `app/globals.css` once.

