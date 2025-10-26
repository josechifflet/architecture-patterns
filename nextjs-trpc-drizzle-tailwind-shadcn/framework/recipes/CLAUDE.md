# Framework Recipes

@../../CLAUDE.md

Step-by-step implementation guides for common tasks.

## Key Recipes

- `add-endpoint.md` - New tRPC procedure (command + router)
- `add-page.md` - New route with RSC and client components
- `add-table.md` - Drizzle schema + migration + seed
- `add-feature.md` - Complete feature (DB → API → UI)

## Recipe Structure

Each recipe includes:
1. **Goal** - What you'll build
2. **Prerequisites** - Dependencies
3. **Steps** - Ordered implementation
4. **Validation** - How to verify
5. **Checklist** - Quality gates

## Common Validation

```bash
pnpm check:types
pnpm lint
pnpm test
./scripts/qa-patterns.sh
make qa
```

See `index.md` for full recipe catalog.
