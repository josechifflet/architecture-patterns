# Naming Conventions

## Files
- Components: `PascalCase.tsx`
- Utilities: `camelCase.ts`
- Pages: `lowercase/page.tsx`
- APIs (route handlers): `camelCase.ts`

## Functions
- tRPC procedures: `verbNoun` (e.g., `getUser`, `createPost`)
- Components: `PascalCase`
- Hooks: `useCamelCase`
- Utilities: `camelCase`

## Database
- Tables: plural (e.g., `users`, `posts`)
- Columns: `snake_case` (e.g., `created_at`)
- Relations: `camelCase` (e.g., `postsByUser`)

## Routers
- One router per feature: `feature.ts` (e.g., `posts.ts`)
- App router aggregator: `_app.ts`

## Zod Schemas
- Input: `<procedureName>Input`
- Output: `<procedureName>Output` (optional if inferred)

[Every naming scenario covered in Patterns and Recipes]

