# Recipe: Add New Database Table

## Prerequisites
- [ ] Drizzle configured with single schema or barrel
- [ ] Database connection configured in `server/db/index.ts`

## Steps

### 1. Create Schema File
File: `src/server/db/schema/[name].ts`
```
import { pgTable, serial, text, timestamp, integer } from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'

export const categories = pgTable('categories', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  created_at: timestamp('created_at').defaultNow().notNull(),
})

export const categoriesRelations = relations(categories, () => ({}))
```

### 2. Add Relations (Optional)
```
import { posts } from './posts'
export const postsRelations = relations(posts, ({ one }) => ({
  category: one(categories, { fields: [posts.category_id], references: [categories.id] })
}))
```

### 3. Generate & Run Migrations
```
npx drizzle-kit generate
npx drizzle-kit migrate
```

### 4. Expose via tRPC (Optional)
File: `src/server/api/routers/categories.ts`
```
import { t } from '@/server/trpc'
import { db } from '@/server/db'
import { categories } from '@/server/db/schema/categories'

export const categoriesRouter = t.router({
  list: t.procedure.query(() => db.select().from(categories)),
})
```

## Verification
- [ ] Table exists and columns match
- [ ] Migration is committed
- [ ] Optional router returns expected data

## Troubleshooting
Problem: Migration not detecting schema file.
Solution: Ensure `drizzle.config.ts` `schema` path includes the new file or barrel.

