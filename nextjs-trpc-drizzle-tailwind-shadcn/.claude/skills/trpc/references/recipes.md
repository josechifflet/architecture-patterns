# Step-By-Step Implementation Recipes

Complete guides for implementing tRPC patterns.

## Recipe 1: Creating a New Router

### Goal

Add a new domain router with list and detail procedures.

### Prerequisites

- Domain entity schema exists in `src/lib/db/schema/<domain>.ts`
- Permission constants defined in `src/lib/auth/permissions.ts`

### Steps

**Step 1: Create command files**

```bash
mkdir -p src/lib/api/commands/queries/<domain>
touch src/lib/api/commands/queries/<domain>/list-<entity>.query.ts
touch src/lib/api/commands/queries/<domain>/get-<entity>.query.ts
```

**Step 2: Implement list query**

```typescript
// src/lib/api/commands/queries/<domain>/list-<entity>.query.ts
import { z } from "zod";
import { createSelectSchema } from "drizzle-zod";
import { and, desc, eq, ilike, count } from "drizzle-orm";

export const ListInputSchema = z
  .object({
    page: z.number().int().min(1).default(1),
    pageSize: z.number().int().min(1).max(100).default(25),
    search: z.string().optional(),
    sortBy: z.enum(["code", "name", "createdAt"]).default("createdAt"),
    sortOrder: z.enum(["asc", "desc"]).default("desc"),
  })
  .strict();

export const ListOutputSchema = z
  .object({
    items: z.array(entitySelectSchema),
    page: z.number().int().min(1),
    pageSize: z.number().int().min(1),
    totalCount: z.number().int().nonnegative(),
    totalPages: z.number().int().nonnegative(),
  })
  .strict();

export type ListInput = z.infer<typeof ListInputSchema>;
export type ListOutput = z.infer<typeof ListOutputSchema>;

export async function listQuery(
  { db }: QueryContext,
  input: ListInput,
): Promise<ListOutput> {
  const { page, pageSize, search, sortBy, sortOrder } = input;

  const whereConditions = [];
  if (search) {
    whereConditions.push(
      or(
        ilike(entities.code, `%${search}%`),
        ilike(entities.name, `%${search}%`),
      ),
    );
  }
  const whereClause =
    whereConditions.length > 0 ? and(...whereConditions) : undefined;

  const sortColumn = entities[sortBy];
  const orderCondition =
    sortOrder === "asc" ? asc(sortColumn) : desc(sortColumn);

  const items = await db
    .select(entityListSelection)
    .from(entities)
    .where(whereClause)
    .orderBy(orderCondition)
    .limit(pageSize)
    .offset((page - 1) * pageSize);

  const [totalResult] = await db
    .select({ count: count() })
    .from(entities)
    .where(whereClause);
  const totalCount = Number(totalResult?.count ?? 0);
  const totalPages = totalCount === 0 ? 0 : Math.ceil(totalCount / pageSize);

  return { items, page, pageSize, totalCount, totalPages };
}
```

**Step 3: Implement detail query**

```typescript
// src/lib/api/commands/queries/<domain>/get-<entity>.query.ts
import { z } from "zod";
import { eq } from "drizzle-orm";
import { NotFoundError } from "@/lib/errors/AppError";

export const GetInputSchema = z.object({ id: z.string().uuid() }).strict();
export const GetOutputSchema = z
  .object({
    id: z.string().uuid(),
    code: z.string(),
    name: z.string(),
    // ... other fields
  })
  .strict();

export type GetInput = z.infer<typeof GetInputSchema>;
export type GetOutput = z.infer<typeof GetOutputSchema>;

export async function getQuery(
  { db }: QueryContext,
  input: GetInput,
): Promise<GetOutput> {
  const [entity] = await db
    .select(entityDetailSelection)
    .from(entities)
    .where(eq(entities.id, input.id))
    .limit(1);

  if (!entity) throw new NotFoundError("Entity");

  return entity;
}
```

**Step 4: Create router file**

```typescript
// src/lib/api/routers/<domain>.router.ts
import { createTRPCRouter, rbacProcedure } from "../trpc";
import {
  ListInputSchema,
  ListOutputSchema,
  listQuery,
} from "../commands/queries/<domain>/list-<entity>.query";
import {
  GetInputSchema,
  GetOutputSchema,
  getQuery,
} from "../commands/queries/<domain>/get-<entity>.query";
import {
  handleMasterDataError,
  assertMasterDataPermission,
} from "./_helpers/master-data";
import { PERMISSIONS } from "@/lib/auth/permissions";

const requireEntityViewPermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.ENTITY_VIEW,
    "Missing view permission",
  );
};

export const entityRouter = createTRPCRouter({
  list: rbacProcedure
    .input(ListInputSchema)
    .output(ListOutputSchema)
    .query(async ({ ctx, input }) => {
      try {
        requireEntityViewPermission(ctx);
        return await listQuery({ db: ctx.db }, input);
      } catch (err) {
        return handleMasterDataError(err, {
          ctx,
          entity: "entity",
          action: "list",
        });
      }
    }),

  get: rbacProcedure
    .input(GetInputSchema)
    .output(GetOutputSchema)
    .query(async ({ ctx, input }) => {
      try {
        requireEntityViewPermission(ctx);
        return await getQuery({ db: ctx.db }, input);
      } catch (err) {
        return handleMasterDataError(err, {
          ctx,
          entity: "entity",
          action: "get",
          payload: input,
        });
      }
    }),
});
```

**Step 5: Register in root router**

```typescript
// src/lib/api/routers/_app.ts
import { entityRouter } from "./entity.router";

export const appRouter = createTRPCRouter({
  // ... existing routers
  entities: entityRouter,
});
```

### Validation

```bash
pnpm check:types                    # TypeScript validation
pnpm lint                           # ESLint checks
./scripts/check-boundaries.sh       # Boundary validation
```

### Checklist

- [ ] Command files export schemas, types, and functions
- [ ] List query returns normalized pagination shape
- [ ] Detail query throws NotFoundError if not found
- [ ] Router uses rbacProcedure for all procedures
- [ ] RBAC helper checks permissions
- [ ] Error handler uses handleMasterDataError
- [ ] Router registered in root router
- [ ] TypeScript types pass validation

## Recipe 2: Adding a Protected Procedure

### Goal

Add a mutation that requires specific permission.

### Prerequisites

- Permission constant defined in `src/lib/auth/permissions.ts`
- Domain error classes available in `src/lib/errors/AppError.ts`

### Steps

**Step 1: Define mutation command**

```typescript
// src/lib/api/commands/mutations/<domain>/<action>.mutation.ts
import { z } from "zod";
import { eq } from "drizzle-orm";
import { NotFoundError } from "@/lib/errors/AppError";

export const ActionInputSchema = z
  .object({
    id: z.string().uuid(),
    // ... action-specific fields
  })
  .strict();

export const ActionOutputSchema = z
  .object({
    id: z.string().uuid(),
    status: z.string(),
    // ... action result fields
  })
  .strict();

export type ActionInput = z.infer<typeof ActionInputSchema>;
export type ActionOutput = z.infer<typeof ActionOutputSchema>;

export async function actionMutation(
  ctx: MutationContext,
  input: ActionInput,
): Promise<ActionOutput> {
  const { db, userId } = ctx;
  const { id } = input;

  return await db.transaction(async (tx) => {
    // Fetch current state
    const [current] = await tx
      .select({ id: entities.id, status: entities.status })
      .from(entities)
      .where(eq(entities.id, id))
      .limit(1);

    if (!current) throw new NotFoundError("Entity");

    // Perform action
    const now = new Date();
    const [updated] = await tx
      .update(entities)
      .set({ status: "new-status", updatedAt: now, updatedBy: userId })
      .where(eq(entities.id, id))
      .returning({ id: entities.id, status: entities.status });

    return { id: updated.id, status: updated.status };
  });
}
```

**Step 2: Add procedure to router**

```typescript
// src/lib/api/routers/<domain>.router.ts
import {
  ActionInputSchema,
  ActionOutputSchema,
  actionMutation,
} from "../commands/mutations/<domain>/<action>.mutation";

const requireEntityManagePermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.ENTITY_MANAGE,
    "Missing manage permission",
  );
};

export const entityRouter = createTRPCRouter({
  // ... existing procedures

  action: rbacProcedure
    .input(ActionInputSchema)
    .output(ActionOutputSchema)
    .mutation(async ({ ctx, input }) => {
      try {
        requireEntityManagePermission(ctx);
        return await actionMutation(ctx, input);
      } catch (err) {
        return handleMasterDataError(err, {
          ctx,
          entity: "entity",
          action: "action",
          payload: input,
        });
      }
    }),
});
```

### Validation

```bash
pnpm check:types
./scripts/qa-patterns.sh
```

### Checklist

- [ ] Mutation command uses transaction for writes
- [ ] Mutation throws domain errors
- [ ] Router uses rbacProcedure
- [ ] Permission check uses assertMasterDataPermission
- [ ] Error handler captures entity/action context

## Recipe 3: Implementing List Queries

### Goal

Create a list query with pagination, search, filters, and sorting.

### Steps

**Step 1: Define input schema with filters**

```typescript
export const ListInputSchema = z
  .object({
    // Pagination (required)
    page: z.number().int().min(1).default(1),
    pageSize: z.number().int().min(1).max(100).default(25),

    // Search (optional)
    search: z.string().optional(),

    // Filters (optional, domain-specific)
    status: statusSchema.optional(),
    category: z.enum(["A", "B", "C"]).optional(),
    isActive: z.boolean().optional(),

    // Sorting (required with defaults)
    sortBy: z.enum(["code", "name", "createdAt"]).default("createdAt"),
    sortOrder: z.enum(["asc", "desc"]).default("desc"),
  })
  .strict();
```

**Step 2: Define output schema with pagination**

```typescript
export const ListOutputSchema = z
  .object({
    items: z.array(entitySelectSchema),
    page: z.number().int().min(1),
    pageSize: z.number().int().min(1),
    totalCount: z.number().int().nonnegative(),
    totalPages: z.number().int().nonnegative(),
  })
  .strict();
```

**Step 3: Implement query with WHERE builder**

```typescript
export async function listQuery(
  { db }: QueryContext,
  input: ListInput,
): Promise<ListOutput> {
  const {
    page,
    pageSize,
    search,
    status,
    category,
    isActive,
    sortBy,
    sortOrder,
  } = input;

  // Build WHERE conditions
  const whereConditions = [];

  if (search) {
    whereConditions.push(
      or(
        ilike(entities.code, `%${search}%`),
        ilike(entities.name, `%${search}%`),
      ),
    );
  }

  if (status) whereConditions.push(eq(entities.status, status));
  if (category) whereConditions.push(eq(entities.category, category));
  if (isActive !== undefined)
    whereConditions.push(eq(entities.isActive, isActive));

  const whereClause =
    whereConditions.length > 0 ? and(...whereConditions) : undefined;

  // Determine sort
  const sortColumn = entities[sortBy];
  const orderCondition =
    sortOrder === "asc" ? asc(sortColumn) : desc(sortColumn);

  // Execute paginated query
  const items = await db
    .select(entityListSelection)
    .from(entities)
    .where(whereClause)
    .orderBy(orderCondition)
    .limit(pageSize)
    .offset((page - 1) * pageSize);

  // Count total
  const [totalResult] = await db
    .select({ count: count() })
    .from(entities)
    .where(whereClause);
  const totalCount = Number(totalResult?.count ?? 0);
  const totalPages = totalCount === 0 ? 0 : Math.ceil(totalCount / pageSize);

  return { items, page, pageSize, totalCount, totalPages };
}
```

### Validation

Verify normalized output shape:

```typescript
// Test or usage
const result = await listQuery({ db }, { page: 1, pageSize: 10 });
assert(result.items);
assert(result.page === 1);
assert(result.pageSize === 10);
assert(typeof result.totalCount === "number");
assert(typeof result.totalPages === "number");
```

### Checklist

- [ ] Input includes page, pageSize, sortBy, sortOrder
- [ ] Output includes items, page, pageSize, totalCount, totalPages
- [ ] WHERE builder uses AND for multiple filters
- [ ] Search uses OR for multiple fields
- [ ] Sort column and order determined from input
- [ ] Count query uses same WHERE clause

## Recipe 4: Mutation with Cache Invalidation

### Goal

Implement mutation that triggers client-side cache invalidation.

### Steps

**Step 1: Implement mutation command**

```typescript
export async function updateMutation(
  ctx: MutationContext,
  input: UpdateInput,
): Promise<UpdateOutput> {
  const { db, userId } = ctx;
  const { id, data } = input;

  return await db.transaction(async (tx) => {
    const [current] = await tx
      .select({ id: entities.id })
      .from(entities)
      .where(eq(entities.id, id))
      .limit(1);

    if (!current) throw new NotFoundError("Entity");

    const now = new Date();
    const [updated] = await tx
      .update(entities)
      .set({ ...data, updatedAt: now, updatedBy: userId })
      .where(eq(entities.id, id))
      .returning({ id: entities.id, code: entities.code, name: entities.name });

    return updated;
  });
}
```

**Step 2: Wire router (no cache hints in command)**

```typescript
update: rbacProcedure
  .input(UpdateInputSchema)
  .output(UpdateOutputSchema)
  .mutation(async ({ ctx, input }) => {
    try {
      requireEntityManagePermission(ctx);
      return await updateMutation(ctx, input);
    } catch (err) {
      return handleMasterDataError(err, {
        ctx,
        entity: "entity",
        action: "update",
        payload: input,
      });
    }
  });
```

**Step 3: Invalidate on client**

```typescript
'use client'
import { api } from '@/lib/api/react'
import { useQueryClient } from '@tanstack/react-query'

export function UpdateEntityButton({ id }: { id: string }) {
  const queryClient = useQueryClient()

  const update = api.entities.update.useMutation({
    onSuccess: (data) => {
      // Invalidate list
      queryClient.invalidateQueries({ queryKey: api.entities.list.queryKey() })

      // Invalidate detail
      queryClient.invalidateQueries({ queryKey: api.entities.get.queryKey({ id }) })

      // Or update detail immediately
      queryClient.setQueryData(api.entities.get.queryKey({ id }), data)
    }
  })

  return <button onClick={() => update.mutate({ id, data: { ... } })}>Update</button>
}
```

### Checklist

- [ ] Mutation command returns domain data only
- [ ] NO invalidation hints in command output
- [ ] Client uses onSuccess to invalidate
- [ ] Invalidate both list and detail queries
- [ ] Consider optimistic updates for better UX

## Recipe 5: Error Handling Recipe

### Goal

Implement comprehensive error handling for a domain.

### Steps

**Step 1: Define domain errors**

```typescript
// src/lib/errors/AppError.ts (add new errors)
export class EntityNotFoundError extends NotFoundError {
  constructor(id: string) {
    super(`Entity with id ${id} not found`);
  }
}

export class InvalidEntityStatusError extends ValidationError {
  constructor(currentStatus: string, requiredStatus: string) {
    super(
      `Entity status is '${currentStatus}' but must be '${requiredStatus}'`,
    );
  }
}
```

**Step 2: Throw domain errors in commands**

```typescript
export async function actionMutation(
  ctx: MutationContext,
  input: ActionInput,
): Promise<ActionOutput> {
  const { db, userId } = ctx;
  const { id } = input;

  return await db.transaction(async (tx) => {
    const [entity] = await tx
      .select({ id: entities.id, status: entities.status })
      .from(entities)
      .where(eq(entities.id, id))
      .limit(1);

    if (!entity) throw new EntityNotFoundError(id);

    if (entity.status !== "expected-status") {
      throw new InvalidEntityStatusError(entity.status, "expected-status");
    }

    // Perform action...
    return result;
  });
}
```

**Step 3: Map errors in router**

```typescript
// Use central error mapper
action: rbacProcedure
  .input(ActionInputSchema)
  .output(ActionOutputSchema)
  .mutation(async ({ ctx, input }) => {
    try {
      return await actionMutation(ctx, input);
    } catch (err) {
      return handleMasterDataError(err, {
        ctx,
        entity: "entity",
        action: "action",
        payload: input,
      });
    }
  });
```

**Step 4: Handle on client**

```typescript
const action = api.entities.action.useMutation({
  onError: (error) => {
    if (error.data?.code === "NOT_FOUND") {
      toast.error("Entity not found");
    } else if (error.data?.code === "BAD_REQUEST") {
      toast.error("Invalid entity status");
    } else {
      toast.error("An unexpected error occurred");
    }
  },
});
```

### Checklist

- [ ] Domain errors extend base classes
- [ ] Commands throw domain errors
- [ ] Routers use central error mapper
- [ ] Errors include entity/action context
- [ ] Client handles specific error codes

## Recipe 6: Nested Router Composition

### Goal

Organize related procedures into nested routers.

### Steps

**Step 1: Create sub-routers**

```typescript
// src/lib/api/routers/stacks/lifecycle.router.ts
export const lifecycleRouter = createTRPCRouter({
  certify: rbacProcedure.input(...).output(...).mutation(...),
  verify: rbacProcedure.input(...).output(...).mutation(...),
  close: rbacProcedure.input(...).output(...).mutation(...),
})

// src/lib/api/routers/stacks/movement.router.ts
export const movementRouter = createTRPCRouter({
  transfer: rbacProcedure.input(...).output(...).mutation(...),
  merge: rbacProcedure.input(...).output(...).mutation(...),
})
```

**Step 2: Compose in main router**

```typescript
// src/lib/api/routers/stacks.router.ts
import { lifecycleRouter } from './stacks/lifecycle.router'
import { movementRouter } from './stacks/movement.router'

export const stacksRouter = createTRPCRouter({
  list: rbacProcedure.input(...).output(...).query(...),
  get: rbacProcedure.input(...).output(...).query(...),

  lifecycle: lifecycleRouter,
  movement: movementRouter,
})
```

**Step 3: Use nested procedures**

```typescript
// Client
const certify = api.stacks.lifecycle.certify.useMutation();
const transfer = api.stacks.movement.transfer.useMutation();

// Server
const caller = await api();
await caller.stacks.lifecycle.certify({ stackId: "..." });
```

### Checklist

- [ ] Sub-routers group related procedures
- [ ] Main router composes sub-routers
- [ ] Nested paths work in client and server

## Recipe 7: Testing Procedures

### Goal

Write integration tests for tRPC procedures.

### Steps

**Step 1: Create test file**

```typescript
// src/lib/api/routers/__tests__/entity.router.test.ts
import { describe, it, expect, beforeEach } from "vitest";
import { createCallerFactory } from "@trpc/server";
import { appRouter } from "../_app";
import { createContext } from "../../context";

const createCaller = createCallerFactory(appRouter);

describe("entity router", () => {
  let caller: ReturnType<typeof createCaller>;

  beforeEach(async () => {
    const ctx = await createContext();
    caller = createCaller(ctx);
  });

  it("lists entities", async () => {
    const result = await caller.entities.list({ page: 1, pageSize: 10 });

    expect(result).toMatchObject({
      items: expect.any(Array),
      page: 1,
      pageSize: 10,
      totalCount: expect.any(Number),
      totalPages: expect.any(Number),
    });
  });

  it("throws NOT_FOUND for non-existent entity", async () => {
    await expect(
      caller.entities.get({ id: "non-existent-id" }),
    ).rejects.toThrow("NOT_FOUND");
  });
});
```

**Step 2: Run tests**

```bash
pnpm test src/lib/api/routers/__tests__/entity.router.test.ts
```

### Checklist

- [ ] Test file in `__tests__` directory
- [ ] Caller created with context
- [ ] Tests verify output shape
- [ ] Tests verify error handling
- [ ] Tests use real database (integration)
