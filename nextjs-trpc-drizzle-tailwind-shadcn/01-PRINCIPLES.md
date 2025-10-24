# Core Principles

## 1. Minimal Files Principle
+ DO: Co-locate related code (UI, procedure calls, styles)
+ DON'T: Split across many tiny files without strong reason

Example:
- Good: `app/posts/page.tsx` uses `server/api/routers/posts.ts` and `server/db/schema/posts.ts`.
- Avoid: scattering helpers across `utils/` with no single owner.

## 2. Pattern Repetition
Every feature follows the same set of patterns:
- Schema → Router → UI
- Queries use the query pattern; Mutations use the mutation pattern
- Naming follows `verbNoun` for procedures and CRUD for routers

## 3. Single Responsibility
Each file does one thing:
- A schema file defines one table
- A router file defines one feature’s procedures
- A component file renders one piece of UI

## 4. Explicit Boundaries
- Server-only logic stays in `src/server/**`
- Client-only code is `"use client"` and isolated
- Route Handlers are only for integration boundaries (webhooks, tRPC endpoint)

## 5. Type-Safety First
- Validate inputs with Zod at router boundary
- Strongly-typed Drizzle schemas and relations
- No `any` in app code

## 6. Single Path to Production
- One way to add a page, component, API, table — documented in Recipes
- If a new need emerges, add/extend a Pattern, then update Recipes

[Each principle is demonstrated concretely in Patterns and Recipes]

