# Next 16 Proxy

Use the Next.js 16 Proxy API to replace legacy middleware for request orchestration (auth, i18n, short-circuits). Keep logic minimal and standards-based (`Request`/`Response`).

## When To Use

- Migrate from `middleware()` and `config.matcher` to Proxy
- Apply global edge guards (auth/i18n) without route handlers
- Short-circuit special paths (`/api`, `/trpc`, SEO files)

## How To

1. Replace default middleware export with `export async function proxy(request: Request)`.
2. Use `new URL(request.url)` and `pathname` checks for selective handling.
3. Guard protected paths via your auth provider (`protect`/redirect).
4. Delegate locale handling to `next-intl` middleware.
5. Optionally set `experimental.skipProxyUrlNormalize: true` in `next.config.ts`.

## Example

```ts
// src/proxy.ts (Next 16)
import createIntlMiddleware from 'next-intl/middleware'
import { auth as clerkAuth, createRouteMatcher } from '@clerk/nextjs/server'
import { routing } from '@/lib/i18n-routing'

const handleI18n = createIntlMiddleware(routing)
const isProtected = createRouteMatcher(['/dashboard(.*)', '/:locale/dashboard(.*)'])

export async function proxy(request: Request): Promise<Response> {
  const url = new URL(request.url)
  const { pathname } = url
  const isApi = pathname.startsWith('/api') || pathname.startsWith('/trpc')
  const isSpecial = pathname === '/monitoring' || pathname === '/robots.txt' || pathname === '/sitemap.xml'

  if (isApi || isSpecial) return new Response(null, { status: 200 })

  const auth = await clerkAuth()
  if (isProtected({ pathname })) {
    await auth.protect({ unauthenticatedUrl: url.origin + '/sign-in' })
  }

  return handleI18n(request)
}
```

```ts
// next.config.ts (optional tweak)
import type { NextConfig } from 'next'

export default {
  experimental: {
    skipProxyUrlNormalize: true,
  },
} satisfies NextConfig
```

## Anti-pattern

- Keeping `middleware()` and `config.matcher` in Next 16
- Spreading proxy logic across multiple ad-hoc route handlers
- Performing heavy logic in the proxy (keep it minimal)

## Related

- routes.md
- server-i18n.md
- ../decisions/2025-10-25-next16-proxy-migration.md
