# tRPC Patterns Reference

Comprehensive pattern catalog for tRPC v11 in Atlas.

## Router Structure and Organization

### File Organization

```
src/lib/api/routers/
  _app.ts                    # Root router combining all domains
  _helpers/
    master-data.ts           # Shared RBAC/error helpers
  stacks.router.ts           # Stack procedures
  audits.router.ts           # Audit procedures
  delivery-notes.router.ts   # Delivery note procedures
  supplier-portal.router.ts  # Supplier portal procedures
  ...
```

### Router Composition

**Root router** (`_app.ts`) combines domain routers:

```typescript
import { createTRPCRouter } from "./trpc";
import { stacksRouter } from "./routers/stacks.router";
import { auditsRouter } from "./routers/audits.router";

export const appRouter = createTRPCRouter({
  stacks: stacksRouter,
  audits: auditsRouter,
  // ... more routers
});

export type AppRouter = typeof appRouter;
```

**Domain router** exports a single router per domain:

```typescript
export const stacksRouter = createTRPCRouter({
  list: rbacProcedure.input(ListInputSchema).output(ListOutputSchema).query(...),
  get: rbacProcedure.input(GetInputSchema).output(GetOutputSchema).query(...),
  certify: rbacProcedure.input(CertifyInputSchema).output(CertifyOutputSchema).mutation(...),
})
```

## Procedure Types

### Query Procedures

Read-only operations using `.query()`:

```typescript
list: rbacProcedure
  .input(ListStacksInputSchema)
  .output(ListStacksOutputSchema)
  .query(async ({ ctx, input }) => {
    try {
      requireStackViewPermission(ctx);
      return await listStacksQuery({ db: ctx.db }, input);
    } catch (err) {
      return handleStackError(err, { ctx, action: "list" });
    }
  });
```

### Mutation Procedures

Write operations using `.mutation()`:

```typescript
certify: rbacProcedure
  .input(CertifyStackInputSchema)
  .output(CertifyStackOutputSchema)
  .mutation(async ({ ctx, input }) => {
    try {
      requireStackManagePermission(ctx);
      return await certifyStackMutation(ctx, input);
    } catch (err) {
      return handleStackError(err, { ctx, action: "certify", payload: input });
    }
  });
```

### Subscription Procedures (Not Used in Atlas)

Atlas doesn't use subscriptions; queries + polling suffice for real-time needs.

## Middleware Patterns

### Logging Middleware

Logs procedure execution:

```typescript
const loggingMiddleware = t.middleware(async ({ ctx, next, path, type }) => {
  const start = Date.now();
  ctx.logger?.debug(`Procedure ${type} ${path} started`);
  try {
    const result = await next();
    const duration = Date.now() - start;
    ctx.logger?.info(`Procedure ${type} ${path} completed`, {
      durationMs: duration,
    });
    return result;
  } catch (error) {
    const duration = Date.now() - start;
    ctx.logger?.error(`Procedure ${type} ${path} failed`, error as Error, {
      durationMs: duration,
    });
    throw error;
  }
});

export const publicProcedure = t.procedure.use(loggingMiddleware);
```

### RBAC Middleware

Enforces authentication and active status:

```typescript
const enforceRBAC = t.middleware(({ ctx, next }) => {
  if (!ctx.userId) {
    throw new TRPCError({
      code: "UNAUTHORIZED",
      message: "Authentication required",
    });
  }
  if (!ctx.dbUserId) {
    throw new TRPCError({
      code: "UNAUTHORIZED",
      message: "User not found in database",
    });
  }
  if (ctx.isInactive) {
    throw new TRPCError({ code: "FORBIDDEN", message: "Account is inactive" });
  }
  return next({
    ctx: {
      ...ctx,
      userId: ctx.userId,
      dbUserId: ctx.dbUserId,
    },
  });
});

export const rbacProcedure = publicProcedure.use(enforceRBAC);
```

### Timing Middleware (Example)

Not currently used, but pattern for performance monitoring:

```typescript
const timingMiddleware = t.middleware(async ({ next }) => {
  const start = performance.now();
  const result = await next();
  const duration = performance.now() - start;
  console.log(`Procedure took ${duration}ms`);
  return result;
});
```

## rbacProcedure Implementation

### Base RBAC

All protected procedures use `rbacProcedure`:

```typescript
export const rbacProcedure = publicProcedure.use(enforceRBAC);
```

This ensures:

- User is authenticated (`userId` present)
- User exists in DB (`dbUserId` present)
- User is active (`!isInactive`)

### Fine-Grained Permissions

Domain-specific permission checks:

```typescript
// In router file
const requireStackManagePermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.STACK_MANAGE,
    "You do not have permission to manage stacks",
  );
};

// In procedure
certify: rbacProcedure
  .input(CertifyStackInputSchema)
  .output(CertifyStackOutputSchema)
  .mutation(async ({ ctx, input }) => {
    try {
      requireStackManagePermission(ctx); // Fine-grained check
      return await certifyStackMutation(ctx, input);
    } catch (err) {
      return handleStackError(err, { ctx, action: "certify", payload: input });
    }
  });
```

### Tenancy Scoping

Supplier portal example with implicit tenancy:

```typescript
async function resolveSupplierIdForUser(
  db: Database,
  userId: string,
): Promise<string> {
  const supplier = await getSupplierByPortalUserIdQuery(
    { db },
    { portalUserId: userId },
  );

  if (!supplier || !supplier.hasPortalAccess) {
    throw new TRPCError({
      code: "NOT_FOUND",
      message: "User is not associated with an active supplier portal account",
    });
  }

  return supplier.id;
}

// In procedure
listDeliveryNotes: rbacProcedure
  .input(listPortalDeliveryNotesQueryInputSchema.omit({ supplierId: true }))
  .output(listPortalDeliveryNotesQueryOutputSchema)
  .query(async ({ ctx, input }) => {
    try {
      requireSupplierPortalViewPermission(ctx);
      const supplierId = await resolveSupplierIdForUser(ctx.db, ctx.userId);
      return await listPortalDeliveryNotesQuery(
        { db: ctx.db },
        { ...input, supplierId },
      );
    } catch (error) {
      return handleSupplierPortalError(error, { ctx, payload: input });
    }
  });
```

## Input Validation with .strict()

### Why .strict()?

Rejects extra properties to prevent accidental data leakage:

```typescript
// Without .strict()
const schema = z.object({ id: z.string() });
schema.parse({ id: "123", password: "secret" }); // ✅ Passes, ignores password

// With .strict()
const schema = z.object({ id: z.string() }).strict();
schema.parse({ id: "123", password: "secret" }); // ❌ Fails, rejects extra properties
```

### Always Use .strict()

```typescript
// ✅ Good
export const ListStacksInputSchema = z.object({
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
}).strict()

list: rbacProcedure
  .input(ListStacksInputSchema) // Already has .strict()
  .output(ListStacksOutputSchema)
  .query(...)

// ❌ Bad
export const ListStacksInputSchema = z.object({
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
})
```

## Output Schemas for Type Safety

### Why Output Schemas?

Ensures commands return exactly what routers expect:

```typescript
export const GetStackOutputSchema = z
  .object({
    id: z.string().uuid(),
    code: z.string(),
    status: stackStatusSchema,
    createdAt: z.string(),
  })
  .strict();

get: rbacProcedure
  .input(GetStackInputSchema)
  .output(GetStackOutputSchema) // Runtime validation
  .query(async ({ ctx, input }) => {
    const stack = await getStackQuery({ db: ctx.db }, input);
    // If stack doesn't match schema, tRPC throws
    return stack;
  });
```

### List Output Schema

All list queries return normalized pagination:

```typescript
export const ListStacksOutputSchema = z
  .object({
    items: z.array(stackSelectSchema),
    page: z.number().int().min(1),
    pageSize: z.number().int().min(1),
    totalCount: z.number().int().nonnegative(),
    totalPages: z.number().int().nonnegative(),
  })
  .strict();
```

## Error Mapping with mapAppErrorToTRPC

### Central Error Mapper

Located in `src/lib/api/trpc-error-mapper.ts`:

```typescript
export function mapAppErrorToTRPC(error: unknown): TRPCError {
  if (error instanceof TRPCError) return error;

  if (error instanceof NotFoundError) {
    return new TRPCError({ code: "NOT_FOUND", message: error.message });
  }

  if (error instanceof ValidationError) {
    return new TRPCError({ code: "BAD_REQUEST", message: error.message });
  }

  if (error instanceof UnauthorizedError) {
    return new TRPCError({ code: "UNAUTHORIZED", message: error.message });
  }

  if (error instanceof ForbiddenError) {
    return new TRPCError({ code: "FORBIDDEN", message: error.message });
  }

  if (error instanceof ConflictError) {
    return new TRPCError({ code: "CONFLICT", message: error.message });
  }

  // Default to internal error
  return new TRPCError({
    code: "INTERNAL_SERVER_ERROR",
    message: "An unexpected error occurred",
  });
}
```

### Domain-Specific Wrappers

Helper functions add observability:

```typescript
export const handleMasterDataError = (
  ctx: PermissionContext,
  {
    entity,
    action,
    payload,
  }: {
    entity: MasterDataEntity;
    action: MasterDataAction;
    payload?: unknown;
  },
  error: unknown,
): never => {
  // Capture in Sentry
  captureMasterDataError(
    {
      entity,
      action,
      actorId: ctx.dbUserId ?? undefined,
      payload,
      requestId: ctx.requestId ?? undefined,
      ipAddress: ctx.ipAddress ?? undefined,
    },
    error,
  );

  // Map and throw
  if (error instanceof TRPCError) throw error;
  if (isAppError(error)) throw mapAppErrorToTRPC(error);

  throw new TRPCError({
    code: "INTERNAL_SERVER_ERROR",
    message: "An unexpected error occurred",
    cause: error,
  });
};
```

## Server Caller Usage in RSC

### Setup

Located in `src/lib/api/server.tsx`:

```typescript
import { createCallerFactory } from "@trpc/server";
import { appRouter } from "./routers/_app";
import { createContext } from "./context";

const createCaller = createCallerFactory(appRouter);

export const api = async () => {
  const ctx = await createContext();
  return createCaller(ctx);
};
```

### Usage in Server Components

```typescript
// app/[locale]/(auth)/dashboard/stacks/page.tsx
import { api } from '@/lib/api/server'

export default async function StacksPage() {
  const caller = await api()
  const stacks = await caller.stacks.list({ page: 1, pageSize: 20 })

  return (
    <div>
      <h1>Stacks ({stacks.totalCount})</h1>
      <ul>
        {stacks.items.map(stack => (
          <li key={stack.id}>{stack.code}</li>
        ))}
      </ul>
    </div>
  )
}
```

### Passing to Client Components

```typescript
// Server component
export default async function StacksPage() {
  const caller = await api()
  const stacks = await caller.stacks.list({ page: 1, pageSize: 20 })

  return <StacksListClient initialData={stacks} />
}

// Client component
'use client'
export function StacksListClient({ initialData }: { initialData: ListStacksOutput }) {
  const { data } = api.stacks.list.useQuery(
    { page: 1, pageSize: 20 },
    { initialData }
  )
  // ...
}
```

## Client Hooks (useQuery, useMutation)

### Setup

Located in `src/lib/api/react.tsx`:

```typescript
import { createTRPCReact } from "@trpc/react-query";
import type { AppRouter } from "./routers/_app";

export const api = createTRPCReact<AppRouter>();
```

### useQuery

```typescript
'use client'
import { api } from '@/lib/api/react'

export function StacksListClient() {
  const { data, isLoading, error } = api.stacks.list.useQuery({
    page: 1,
    pageSize: 20,
  })

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>

  return (
    <div>
      {data.items.map(stack => (
        <div key={stack.id}>{stack.code}</div>
      ))}
    </div>
  )
}
```

### useMutation

```typescript
'use client'
import { api } from '@/lib/api/react'
import { useQueryClient } from '@tanstack/react-query'

export function CertifyStackButton({ stackId }: { stackId: string }) {
  const queryClient = useQueryClient()

  const certify = api.stacks.certify.useMutation({
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: api.stacks.list.queryKey() })
      queryClient.invalidateQueries({ queryKey: api.stacks.get.queryKey({ id: stackId }) })
    },
    onError: (error) => {
      console.error('Certification failed:', error.message)
    }
  })

  return (
    <button onClick={() => certify.mutate({ stackId })}>
      {certify.isPending ? 'Certifying...' : 'Certify Stack'}
    </button>
  )
}
```

### Optimistic Updates

```typescript
const updateStack = api.stacks.update.useMutation({
  onMutate: async (newStack) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({
      queryKey: api.stacks.get.queryKey({ id: newStack.id }),
    });

    // Snapshot previous value
    const previous = queryClient.getQueryData(
      api.stacks.get.queryKey({ id: newStack.id }),
    );

    // Optimistically update
    queryClient.setQueryData(
      api.stacks.get.queryKey({ id: newStack.id }),
      newStack,
    );

    return { previous };
  },
  onError: (err, newStack, context) => {
    // Rollback on error
    queryClient.setQueryData(
      api.stacks.get.queryKey({ id: newStack.id }),
      context?.previous,
    );
  },
  onSettled: (newStack) => {
    // Refetch after success or error
    queryClient.invalidateQueries({
      queryKey: api.stacks.get.queryKey({ id: newStack.id }),
    });
  },
});
```

## Cache Invalidation Patterns

### Invalidate Affected Queries

After mutation, invalidate related queries:

```typescript
const certify = api.stacks.certify.useMutation({
  onSuccess: (data) => {
    // Invalidate list (stack appears in list)
    queryClient.invalidateQueries({ queryKey: api.stacks.list.queryKey() });

    // Invalidate detail (stack status changed)
    queryClient.invalidateQueries({
      queryKey: api.stacks.get.queryKey({ id: data.stackId }),
    });

    // Invalidate related entities
    queryClient.invalidateQueries({ queryKey: api.audits.list.queryKey() });
  },
});
```

### Specific Query Key

```typescript
// Invalidate all stack lists
queryClient.invalidateQueries({ queryKey: api.stacks.list.queryKey() });

// Invalidate specific stack detail
queryClient.invalidateQueries({
  queryKey: api.stacks.get.queryKey({ id: stackId }),
});

// Invalidate all stacks queries
queryClient.invalidateQueries({ queryKey: ["stacks"] });
```

### setQueryData for Immediate Updates

```typescript
const certify = api.stacks.certify.useMutation({
  onSuccess: (data) => {
    // Update detail immediately
    queryClient.setQueryData(
      api.stacks.get.queryKey({ id: data.stackId }),
      (old) => ({ ...old, status: "certified", certifiedAt: data.certifiedAt }),
    );

    // Still invalidate list to refetch
    queryClient.invalidateQueries({ queryKey: api.stacks.list.queryKey() });
  },
});
```

## Type Inference

### RouterOutputs

Derive types from router outputs:

```typescript
import type { RouterOutputs } from "@/lib/api/root";

// List item type
type Stack = RouterOutputs["stacks"]["list"]["items"][number];

// Single entity type
type StackDetail = RouterOutputs["stacks"]["get"];

// Mutation output
type CertifyResult = RouterOutputs["stacks"]["certify"];
```

### RouterInputs

Derive input types:

```typescript
import type { RouterInputs } from "@/lib/api/root";

type ListStacksInput = RouterInputs["stacks"]["list"];
type CertifyStackInput = RouterInputs["stacks"]["certify"];
```

### Command Types

Commands export inferred types:

```typescript
// In command file
export const ListStacksInputSchema = z.object({ ... }).strict()
export const ListStacksOutputSchema = z.object({ ... }).strict()

export type ListStacksInput = z.infer<typeof ListStacksInputSchema>
export type ListStacksOutput = z.infer<typeof ListStacksOutputSchema>
```

UI can import command types directly if needed, but prefer RouterOutputs.
