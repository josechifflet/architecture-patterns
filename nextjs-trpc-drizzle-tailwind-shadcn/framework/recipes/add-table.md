# Add Table

Create a new PostgreSQL table with Drizzle, generate a migration, and run first queries.

## Prerequisites

- Drizzle configured and DB reachable
- Migrations directory set up

## Steps

1. Define the schema

```ts
// src/lib/db/schema/projects.ts
import { pgTable, serial, text, timestamp, index } from 'drizzle-orm/pg-core'

export const projects = pgTable(
  'projects',
  {
    id: serial('id').primaryKey(),
    name: text('name').notNull(),
    created_at: timestamp('created_at').defaultNow(),
  },
  (table) => ({ nameIdx: index('projects_name_idx').on(table.name) })
)
```

2. Generate and apply a migration

```bash
# generate SQL from schema changes
pnpm drizzle-kit generate

# apply migrations
pnpm drizzle-kit migrate
```

3. First queries

```ts
// src/lib/db/examples/projects.ts
import { db } from '@/lib/db'
import { projects } from '@/lib/db/schema/projects'

// Insert
await db.insert(projects).values({ name: 'Alpha' })

// Select
const all = await db.select().from(projects)

// Update
// await db.update(projects).set({ name: 'Beta' }).where(eq(projects.id, 1))
```

## Verification

- Migration file created under `migrations/`
- Table appears in the database and accepts inserts
- Queries compile with inferred types
- i18n: Not applicable (no UI). If adding UI later, use `getTranslations()` in RSC pages and `useTranslations()` in client components [see patterns/server-i18n.md]

## Related

- patterns/drizzle-schemas.md
- patterns/queries.md
 - patterns/server-i18n.md
