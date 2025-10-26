# Framework Patterns

@../../CLAUDE.md

22 implementation patterns solving recurring problems.

## Critical Patterns

1. `server-components.md` - RSC data fetching
2. `client-components.md` - Interactive UI
3. `trpc-procedures.md` - Router/command boundary
4. `queries.md` - List normalization
5. `mutations.md` - Mutation shape + caching
6. `error-handling.md` - Central error mapping

## Full Index

See `index.md` for complete list of 22 patterns covering:
- Components (RSC, client, forms, layouts, providers)
- API (tRPC, queries, mutations, errors, cache)
- Data (Drizzle, RSC prefetch/hydration)
- Styling (Tailwind v4, shadcn)
- Infrastructure (routes, proxy, i18n)

## Enforcement

- `./scripts/qa-patterns.sh` - Validates RBAC, list queries, error mapping
- `./scripts/check-boundaries.sh` - No DB/commands in UI
- ESLint custom rules
- Contract tests
