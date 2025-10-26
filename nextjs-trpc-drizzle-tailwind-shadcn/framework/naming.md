# Naming

Use consistent names across files, functions, components, tables, and procedures. Favor clarity and predictability over brevity.

## Files and Folders

- Filenames and folders: kebab-case
- React components: PascalCase.tsx
- Commands: `<action>.mutation.ts` or `list-<items>.query.ts`
- Routers: `<domain>.router.ts`
- Drizzle schema files: `<table>.ts` in snake_case table names

## Routes

- Route groups: `(group)`
- Dynamic segments: `[id]`, `[slug]`, `[...catchAll]`
- Files: `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`

## Components

- Presentational components: PascalCase with props‑only APIs
- Interactive containers: end with `Client` suffix
- UI primitives: live under `src/components/ui/*` with their semantic name
- UI types: `src/lib/types/ui/<feature>.ts`

## tRPC

- Routers: plural domain, e.g., `tasksRouter`
- Procedures:
  - Queries: `list`, `get`, `byId`, `search`
  - Mutations: `create`, `update`, `delete`, `archive`
- Input schemas: `<Action>InputSchema`
- Output schemas: `<Action>OutputSchema`

## Database (Drizzle)

- Tables: snake_case, plural (e.g., `tasks`)
- Columns: snake_case (e.g., `created_at`)
- Indexes: `<table>_<column>_idx`
- Relations: explicit column FKs; name unique constraints as needed

## Tests

- Unit/integration: `*.test.ts` or `*.test.tsx`
- E2E: `tests/e2e/<feature>.spec.ts`
- Stable selectors: `data-testid="<feature-element>"`
- QA commands: `make qa` before PRs

Consistent names make code searchable and patterns easy to repeat.
