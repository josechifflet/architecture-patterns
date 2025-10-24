# Route Handlers

## When to Use
- Integration boundaries: webhooks, revalidation, tRPC endpoint
- Non-UI concerns under `app/api/**/route.ts`

## Structure
```
app/api/
  trpc/[trpc]/route.ts       # tRPC
  revalidate/route.ts        # POST revalidateTag
```

## Complete Example
Basic GET:
```ts
// app/api/ping/route.ts
export function GET() { return new Response('pong') }
```

tRPC endpoint:
```ts
// app/api/trpc/[trpc]/route.ts
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@/server/api/routers/_app'
import { createContext } from '@/server/trpc'

const handler = (req: Request) =>
  fetchRequestHandler({ endpoint: '/api/trpc', req, router: appRouter, createContext })

export { handler as GET, handler as POST }
```

## Anti-Patterns
❌ Large business logic inside handlers.
✅ Delegate to routers/services.

## Checklist
- [ ] Uses Web Request/Response API
- [ ] Thin handler delegates to server code
- [ ] Typed responses when relevant

## Related Patterns
- See: `../data/trpc-procedures.md`

