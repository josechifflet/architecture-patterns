# Cache & Revalidation

Explicitly declare cache intent in RSC and use tags for on-demand revalidation.

## When To Use

- SSG/ISR for public data
- Per-request freshness for user-specific data
- Webhook-driven cache busting

## Fetch with Tags

```ts
// RSC
const data = await fetch('https://api.example.com/posts', {
  next: { revalidate: 60, tags: ['posts'] },
}).then((r) => r.json())
```

## Revalidate by Tag

```ts
// app/api/revalidate/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { revalidateTag } from 'next/cache'

export async function POST(req: NextRequest) {
  const secret = req.headers.get('x-reval-secret')
  if (secret !== process.env.REVALIDATE_SECRET) {
    return NextResponse.json({ ok: false }, { status: 401 })
  }
  revalidateTag('posts')
  return NextResponse.json({ ok: true })
}
```

## Notes

- Pair route handlers with upstream webhook signatures for security.
- Prefer tags for targeted invalidation over global revalidate.

## How To

1. Fetch in RSC with `next: { revalidate, tags }` to opt into ISR + tags.
2. Expose a small route handler that calls `revalidateTag('tag')` on trusted events.
3. Protect the handler with a shared secret or signature verification.
4. Use multiple tags to target different lists or details as needed.

## Anti-pattern

- Calling global revalidation on every mutation instead of tag-scoped invalidation
- Exposing revalidation endpoints without a secret/signature
- Mixing tag names between features (use stable, feature-scoped names)
