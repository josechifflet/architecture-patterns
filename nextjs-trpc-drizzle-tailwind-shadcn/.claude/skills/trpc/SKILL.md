---
name: trpc
description: tRPC v11 patterns for Next.js applications with command-query separation. This skill should be used when creating routers, procedures, implementing RBAC, handling errors, or using server caller patterns. Covers rbacProcedure, input validation, output schemas, error mapping, cache invalidation, and type inference.
---

# tRPC v11 Patterns for Next.js Applications

## Purpose

Provide comprehensive tRPC v11 implementation patterns for Next.js 16 applications emphasizing thin routers with command-query separation. Focus on RBAC enforcement, central error mapping, type safety through inference, and proper client/server boundaries.

## When To Use This Skill

**Router & Procedure Creation:**
- Create new tRPC routers for domain entities
- Implement query procedures for data fetching
- Implement mutation procedures for state changes
- Set up public vs protected (RBAC) procedures

**Validation & Type Safety:**
- Define Zod input schemas with .strict() validation
- Define Zod output schemas for type inference
- Export RouterInputs/RouterOutputs types for UI
- Implement type-safe server callers for RSC

**RBAC & Security:**
- Enforce authentication with rbacProcedure
- Implement fine-grained permission checks
- Validate user roles and permissions
- Handle authorization failures consistently

**Error Handling:**
- Map domain errors to tRPC error codes
- Use central error mapping (avoid inline TRPCError)
- Handle NotFoundError, ValidationError, ConflictError
- Capture errors with proper context for Sentry

**Client Integration:**
- Use tRPC server caller in Server Components
- Set up React Query hooks in Client Components
- Implement optimistic updates
- Configure cache invalidation strategies

**Architecture:**
- Separate business logic into commands
- Keep routers thin (validate → delegate → map errors)
- Maintain clear server/client boundaries
- Follow command-query separation principle

## Core Architecture

### Separation of Concerns

```
UI Component → tRPC Hook → Router → Command → Database
                ↓             ↓        ↓
           Type Safety   Validation  Logic
                         RBAC        Errors
                         Error Map
```

**Commands** contain pure business logic:

- Export Zod input/output schemas
- Accept typed input (already validated)
- Throw domain errors
- Return typed domain data
- NO parsing, NO RBAC, NO TRPCError

**Routers** are thin orchestrators:

- Validate with `.input(Schema.strict())`
- Enforce RBAC with `rbacProcedure`
- Delegate to command
- Map domain errors to TRPC errors

### Key Principles

1. **Thin Routers** - Validate, enforce RBAC, delegate, map errors
2. **RBAC at Boundary** - Permission checks ONLY in routers
3. **Central Error Mapping** - NO inline `new TRPCError()`
4. **Strict Validation** - Always `.input(Schema.strict())`
5. **Command Delegation** - Business logic lives in commands

## Procedures and Middleware

### Procedure Types

**publicProcedure** - No auth required (health, landing pages)

```typescript
export const publicProcedure = t.procedure.use(loggingMiddleware);
```

**rbacProcedure** - Protected, enforces auth + active status

```typescript
export const rbacProcedure = publicProcedure.use(enforceRBAC);
```

### RBAC Middleware

```typescript
const enforceRBAC = t.middleware(({ ctx, next }) => {
  if (!ctx.userId) throw new TRPCError({ code: "UNAUTHORIZED" });
  if (!ctx.dbUserId) throw new TRPCError({ code: "UNAUTHORIZED" });
  if (ctx.isInactive) throw new TRPCError({ code: "FORBIDDEN" });
  return next({ ctx: { ...ctx, userId: ctx.userId, dbUserId: ctx.dbUserId } });
});
```

Fine-grained permissions use helpers:

```typescript
const requirePermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.HAULER_VIEW,
    "Missing permission",
  );
};
```

### Context Structure

Available in all procedures:

- `userId` - Clerk user ID (string)
- `dbUserId` - Internal user UUID (string)
- `userRole` - Single role (legacy)
- `userRoles` - Array of roles (current)
- `isInactive` - User deactivated flag
- `db` - Drizzle client
- `logger` - Pino logger
- `requestId`, `ipAddress` - Request metadata

## Implementing Procedures

### List Query Pattern

**Step 1: Define Command** (`src/lib/api/commands/queries/<domain>/list-<entity>.query.ts`)

```typescript
import { z } from "zod";
import { createSelectSchema } from "drizzle-zod";
import { and, desc, eq, ilike, count } from "drizzle-orm";

// Input schema with pagination, filters, sort
export const ListHaulersInputSchema = z
  .object({
    page: z.number().int().min(1).default(1),
    pageSize: z.number().int().min(1).max(100).default(25),
    search: z.string().optional(),
    status: haulerStatusSchema.optional(),
    sortBy: z.enum(["code", "name", "createdAt"]).default("createdAt"),
    sortOrder: z.enum(["asc", "desc"]).default("desc"),
  })
  .strict();

// Output schema - MUST include normalized pagination shape
export const ListHaulersOutputSchema = z
  .object({
    items: z.array(haulerSelectSchema),
    page: z.number().int().min(1),
    pageSize: z.number().int().min(1),
    totalCount: z.number().int().nonnegative(),
    totalPages: z.number().int().nonnegative(),
  })
  .strict();

export type ListHaulersInput = z.infer<typeof ListHaulersInputSchema>;
export type ListHaulersOutput = z.infer<typeof ListHaulersOutputSchema>;

export async function listHaulersQuery(
  { db }: QueryContext,
  input: ListHaulersInput,
): Promise<ListHaulersOutput> {
  const { page, pageSize, search, status, sortBy, sortOrder } = input;

  // Build where conditions
  const whereConditions = [];
  if (search) {
    whereConditions.push(
      or(
        ilike(haulers.code, `%${search}%`),
        ilike(haulers.name, `%${search}%`),
      ),
    );
  }
  if (status) whereConditions.push(eq(haulers.status, status));
  const whereClause =
    whereConditions.length > 0 ? and(...whereConditions) : undefined;

  // Determine sort
  const sortColumn = haulers[sortBy];
  const orderCondition =
    sortOrder === "asc" ? asc(sortColumn) : desc(sortColumn);

  // Execute paginated query
  const items = await db
    .select(haulerListSelection) // Explicit column selection
    .from(haulers)
    .where(whereClause)
    .orderBy(orderCondition)
    .limit(pageSize)
    .offset((page - 1) * pageSize);

  // Count total
  const [totalResult] = await db
    .select({ count: count() })
    .from(haulers)
    .where(whereClause);
  const totalCount = Number(totalResult?.count ?? 0);
  const totalPages = totalCount === 0 ? 0 : Math.ceil(totalCount / pageSize);

  return { items, page, pageSize, totalCount, totalPages };
}
```

**Step 2: Wire Router** (`src/lib/api/routers/<domain>.router.ts`)

```typescript
export const haulersRouter = createTRPCRouter({
  list: rbacProcedure
    .input(ListHaulersInputSchema)
    .output(ListHaulersOutputSchema)
    .query(async ({ ctx, input }) => {
      try {
        requireMasterDataViewPermission(ctx);
        return await listHaulersQuery({ db: ctx.db }, input);
      } catch (err) {
        return handleMasterDataError(err, {
          ctx,
          entity: "hauler",
          action: "list",
        });
      }
    }),
});
```

### Mutation Pattern

**Step 1: Define Command** (`src/lib/api/commands/mutations/<domain>/<action>.mutation.ts`)

```typescript
import { z } from "zod";
import { eq } from "drizzle-orm";
import { NotFoundError, InvalidStackStatusError } from "@/lib/errors/AppError";

export const CertifyStackInputSchema = z
  .object({
    stackId: z.string().uuid(),
    override: z
      .object({
        reason: z.string().min(10).max(500),
      })
      .optional(),
  })
  .strict();

export const CertifyStackOutputSchema = z
  .object({
    stackId: z.string().uuid(),
    status: z.literal("certified"),
    certifiedAt: z.string(),
  })
  .strict();

export type CertifyStackInput = z.infer<typeof CertifyStackInputSchema>;
export type CertifyStackOutput = z.infer<typeof CertifyStackOutputSchema>;

export async function certifyStackMutation(
  ctx: MutationContext,
  input: CertifyStackInput,
): Promise<CertifyStackOutput> {
  const { db, userId } = ctx;
  const { stackId, override } = input;

  return await db.transaction(async (tx) => {
    // Fetch current state
    const [currentStack] = await tx
      .select({
        id: stacks.id,
        status: stacks.status,
        verifiedBy: stacks.verifiedBy,
      })
      .from(stacks)
      .where(eq(stacks.id, stackId))
      .limit(1);

    if (!currentStack) throw new NotFoundError("Stack");
    if (currentStack.status !== "verified") {
      throw new InvalidStackStatusError(currentStack.status, "verified");
    }

    // Enforce SoD
    if (currentStack.verifiedBy === userId) {
      throw new SeparationOfDutiesViolationError("certify", "verify", userId);
    }

    // Update stack
    const now = new Date();
    const [certifiedStack] = await tx
      .update(stacks)
      .set({
        status: "certified",
        certifiedAt: now,
        certifiedBy: userId,
        updatedAt: now,
        updatedBy: userId,
      })
      .where(eq(stacks.id, stackId))
      .returning({
        id: stacks.id,
        status: stacks.status,
        certifiedAt: stacks.certifiedAt,
      });

    // Audit trail
    await tx.insert(stackAudits).values({
      stackId,
      action: "certified",
      actorUserId: userId,
      occurredAt: now,
      beforeJsonb: { status: "verified" },
      afterJsonb: { status: "certified", certifiedAt: now.toISOString() },
    });

    return {
      stackId: certifiedStack.id,
      status: "certified",
      certifiedAt: certifiedStack.certifiedAt!.toISOString(),
    };
  });
}
```

**Step 2: Wire Router**

```typescript
export const stacksRouter = createTRPCRouter({
  certify: rbacProcedure
    .input(CertifyStackInputSchema)
    .output(CertifyStackOutputSchema)
    .mutation(async ({ ctx, input }) => {
      try {
        requireStackManagePermission(ctx);
        return await certifyStackMutation(ctx, input);
      } catch (err) {
        return handleStackError(err, {
          ctx,
          action: "certify",
          payload: input,
        });
      }
    }),
});
```

## Error Handling

### Domain Errors

Commands throw domain-specific errors from `src/lib/errors/AppError.ts`:

- `NotFoundError` - Entity not found
- `ValidationError` - Business rule violation
- `InvalidStackStatusError` - Invalid state transition
- `SeparationOfDutiesViolationError` - SoD violation
- `ConflictError` - Duplicate entity

### Central Error Mapping

Routers use `mapAppErrorToTRPC` (or domain-specific wrappers):

```typescript
// Generic mapping
try {
  return await command(ctx, input);
} catch (err) {
  throw mapAppErrorToTRPC(err);
}

// Domain-specific wrapper
return handleMasterDataError(err, {
  ctx,
  entity: "hauler",
  action: "create",
  payload: input,
});
```

**Never throw inline TRPCError**:

```typescript
// ❌ Bad - inline error
throw new TRPCError({ code: 'NOT_FOUND', message: 'Stack not found' })

// ✅ Good - domain error in command
throw new NotFoundError('Stack')

// ✅ Good - mapped in router
catch (err) { throw mapAppErrorToTRPC(err) }
```

## Client Usage

### Server Caller in RSC

Server components use direct async calls:

```typescript
// app/[locale]/(auth)/dashboard/stacks/page.tsx
import { api } from '@/lib/api/server'

export default async function StacksPage() {
  const stacks = await api().stacks.list({ page: 1, pageSize: 20 })

  return (
    <div>
      <h1>Stacks ({stacks.totalCount})</h1>
      {stacks.items.map(stack => (
        <div key={stack.id}>{stack.code}</div>
      ))}
    </div>
  )
}
```

### React Query Hooks

Client components use tRPC hooks:

```typescript
'use client'
import { api } from '@/lib/api/react'

export function StacksListClient() {
  const { data, isLoading } = api.stacks.list.useQuery({ page: 1, pageSize: 20 })

  if (isLoading) return <div>Loading...</div>

  return (
    <div>
      {data.items.map(stack => (
        <div key={stack.id}>{stack.code}</div>
      ))}
    </div>
  )
}
```

### Mutations with Cache Invalidation

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
    }
  })

  return (
    <button onClick={() => certify.mutate({ stackId })}>Certify</button>
  )
}
```

## Type Inference

### RouterOutputs

UI derives types from router outputs:

```typescript
import type { RouterOutputs } from "@/lib/api/root";

// Infer list item type
type Stack = RouterOutputs["stacks"]["list"]["items"][number];

// Infer single entity type
type StackDetail = RouterOutputs["stacks"]["get"];
```

**Never import DB types in UI**:

```typescript
// ❌ Bad - importing DB schema
import type { Stack } from "@/lib/db/schema";

// ✅ Good - deriving from router
type Stack = RouterOutputs["stacks"]["list"]["items"][number];
```

## Common Mistakes

1. **Validating in Commands** - Commands receive pre-validated input
2. **RBAC in Commands** - Permission checks belong in routers
3. **Inline TRPCError** - Always use central error mapping
4. **Missing .strict()** - Reject extra properties with `.input(Schema.strict())`
5. **Leaking DB Types** - UI derives from RouterOutputs
6. **Inconsistent List Shape** - Always return `{ items, page, pageSize, totalCount, totalPages }`
7. **Cache Hints in Commands** - Commands return domain data only

## Quick Templates

### Router Procedure

```typescript
export const domainRouter = createTRPCRouter({
  action: rbacProcedure
    .input(ActionInputSchema)
    .output(ActionOutputSchema)
    .mutation(async ({ ctx, input }) => {
      try {
        requirePermission(ctx);
        return await actionCommand(ctx, input);
      } catch (err) {
        return handleDomainError(err, {
          ctx,
          action: "action",
          payload: input,
        });
      }
    }),
});
```

### Command

```typescript
export const ActionInputSchema = z
  .object({
    /* ... */
  })
  .strict();
export const ActionOutputSchema = z
  .object({
    /* ... */
  })
  .strict();
export type ActionInput = z.infer<typeof ActionInputSchema>;
export type ActionOutput = z.infer<typeof ActionOutputSchema>;

export async function actionCommand(
  ctx: MutationContext,
  input: ActionInput,
): Promise<ActionOutput> {
  // Business logic here
  return result;
}
```

## Resources

See `references/` for detailed pattern documentation:

- `patterns.md` - Comprehensive pattern reference
- `recipes.md` - Step-by-step implementation guides
- `examples.md` - Full working examples

Codebase references:

- `src/lib/api/trpc.ts` - tRPC setup
- `src/lib/api/routers/` - Router implementations
- `src/lib/api/commands/` - Command implementations
- `src/lib/errors/AppError.ts` - Domain errors
- `framework/patterns/trpc-procedures.md` - Framework docs
