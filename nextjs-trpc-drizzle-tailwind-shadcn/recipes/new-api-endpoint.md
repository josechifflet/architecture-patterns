# Recipe: Add New API Endpoint

## Prerequisites
- [ ] Purpose is integration boundary (not UI data)
- [ ] Consider if it should be a tRPC procedure instead

## Steps

### 1. Create Route Handler
File: `src/app/api/[name]/route.ts`
```
export async function POST(req: Request) {
  const body = await req.json()
  // validate body
  return Response.json({ ok: true })
}
```

### 2. Add tRPC Endpoint (if not existing)
File: `src/app/api/trpc/[trpc]/route.ts`
```
import { fetchRequestHandler } from '@trpc/server/adapters/fetch'
import { appRouter } from '@/server/api/routers/_app'
import { createContext } from '@/server/trpc'

const handler = (req: Request) =>
  fetchRequestHandler({ endpoint: '/api/trpc', req, router: appRouter, createContext })

export { handler as GET, handler as POST }
```

## Verification
- [ ] `curl` returns expected JSON
- [ ] Errors mapped to proper status codes

## Troubleshooting
Problem: 401 from revalidate endpoint.
Solution: Ensure secret header matches and use `revalidateTag` correctly inside POST handler.

