# Drizzle Schemas

Define PostgreSQL tables with typed columns, indexes, and relations. Keep names snake_case and colocate one table per file.

## When To Use

- Adding or changing database structure
- Declaring relations and constraints
- Enabling type inference for queries

## Basic Structure

```ts
// src/lib/db/schema/tasks.ts
import { pgTable, serial, text, boolean, timestamp, index } from 'drizzle-orm/pg-core'
import type { InferSelectModel, InferInsertModel } from 'drizzle-orm'

export const tasks = pgTable(
  'tasks',
  {
    id: serial('id').primaryKey(),
    title: text('title').notNull(),
    done: boolean('done').notNull().default(false),
    created_at: timestamp('created_at', { withTimezone: true })
      .notNull()
      .defaultNow(),
    updated_at: timestamp('updated_at', { withTimezone: true })
      .notNull()
      .defaultNow()
      .$onUpdate(() => new Date()),
  },
  (t) => [
    // Index for common queries
    index('tasks_title_idx').on(t.title),
    index('tasks_done_idx').on(t.done),
    index('tasks_created_at_idx').on(t.created_at),
  ]
)

// Export type-safe models
export type Task = InferSelectModel<typeof tasks>
export type NewTask = InferInsertModel<typeof tasks>

// Optionally export partial types for updates
export type TaskUpdate = Partial<Omit<Task, 'id' | 'created_at'>>
```

## How To

1. Create one table per file under `src/lib/db/schema/<table>.ts` with snake_case names.
2. Define columns, indexes, uniques, and relations explicitly; export the table.
3. Generate and run migrations via Drizzle Kit to sync SQL.
4. Import tables in commands/queries; do not import schema from UI.

## Anti-pattern

- Multiple tables per file (harder to search and change)
- CamelCase table/column names (breaks convention)
- Embedding queries in schema files (mixes concerns)
- Importing schema from any UI code

## Complete Example

```ts
// src/lib/db/schema/users.ts
import { pgTable, serial, text, timestamp, uniqueIndex } from 'drizzle-orm/pg-core'
import type { InferSelectModel, InferInsertModel } from 'drizzle-orm'

export const users = pgTable(
  'users',
  {
    id: serial('id').primaryKey(),
    name: text('name').notNull(),
    email: text('email').notNull(),
    created_at: timestamp('created_at', { withTimezone: true })
      .notNull()
      .defaultNow(),
  },
  (t) => [uniqueIndex('users_email_idx').on(t.email)]
)

// Export type-safe models
export type User = InferSelectModel<typeof users>
export type NewUser = InferInsertModel<typeof users>
```

```ts
// src/lib/db/schema/tasks.ts
import { pgTable, serial, text, boolean, integer, timestamp, index } from 'drizzle-orm/pg-core'
import { relations } from 'drizzle-orm'
import type { InferSelectModel, InferInsertModel } from 'drizzle-orm'
import { users } from './users'

export const tasks = pgTable(
  'tasks',
  {
    id: serial('id').primaryKey(),
    title: text('title').notNull(),
    done: boolean('done').notNull().default(false),
    author_id: integer('author_id')
      .notNull()
      .references(() => users.id, { onDelete: 'cascade' }),
    created_at: timestamp('created_at', { withTimezone: true })
      .notNull()
      .defaultNow(),
    updated_at: timestamp('updated_at', { withTimezone: true })
      .notNull()
      .defaultNow()
      .$onUpdate(() => new Date()),
  },
  (t) => [
    index('tasks_title_idx').on(t.title),
    index('tasks_author_id_idx').on(t.author_id),
    index('tasks_done_idx').on(t.done),
  ]
)

// Type-safe relations
export const tasksRelations = relations(tasks, ({ one }) => ({
  author: one(users, {
    fields: [tasks.author_id],
    references: [users.id],
  }),
}))

// Export type-safe models
export type Task = InferSelectModel<typeof tasks>
export type NewTask = InferInsertModel<typeof tasks>

// Type for tasks with joined author
export type TaskWithAuthor = Task & {
  author: {
    id: number
    name: string
    email: string
  }
}
```

## Common Mistakes

- CamelCase table or column names; use snake_case
- Missing indexes on frequent filters or joins
- Foreign keys without on delete policy when required by invariants

Notes

- Use `count(table.id)` from `drizzle-orm` for totals and aggregations; coerce bigint to number explicitly when needed: `Number(total)`.
- Prefer pooled connections for production (e.g., Neon/Vercel) and wrap multi-step writes in transactions for invariants.

## Related

- queries.md
- mutations.md
