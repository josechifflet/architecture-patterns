# Drizzle Schemas

## When to Use
- Define one table per file under `src/server/db/schema`
- Encode constraints and relations in TypeScript

## Structure
```
server/db/
  schema/
    posts.ts
  index.ts            # db client
```

## Complete Example
File: `src/server/db/schema/posts.ts`
```ts
import { pgTable, serial, text, timestamp } from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  created_at: timestamp('created_at').defaultNow().notNull(),
})

export const postsRelations = relations(posts, () => ({}))
```

File: `drizzle.config.ts` (single schema file)
```ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/server/db/schema/posts.ts', // or a barrel file
  out: './drizzle',
})
```

Migrations:
```bash
npx drizzle-kit generate
npx drizzle-kit migrate
```

## Anti-Patterns
❌ Cross-import tables in complex cycles.
✅ Keep each table in its own file; export a barrel if needed.

## Checklist
- [ ] One table per file
- [ ] Snake_case columns
- [ ] Primary key named `id` unless strong reason
- [ ] All NOT NULL and defaults encoded

## Related Patterns
- See: `queries.md`, `mutations.md`
- See: `recipes/new-database-table.md`

