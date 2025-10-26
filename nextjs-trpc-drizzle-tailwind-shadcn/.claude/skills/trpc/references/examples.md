# Complete tRPC Examples

Full working implementations demonstrating tRPC patterns in Atlas.

## Example 1: Complete CRUD Router

### Haulers Router

Full CRUD implementation with list, detail, create, update, and delete.

#### Schema (Drizzle)

```typescript
// src/lib/db/schema/haulers.ts
import { pgTable, uuid, text, timestamp } from "drizzle-orm/pg-core";

export const haulers = pgTable("haulers", {
  id: uuid("id").primaryKey().defaultRandom(),
  code: text("code").notNull().unique(),
  name: text("name").notNull(),
  legalName: text("legal_name").notNull(),
  status: text("status").notNull().default("active"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
  createdBy: uuid("created_by").references(() => users.id),
  updatedBy: uuid("updated_by").references(() => users.id),
});

export const haulerStatusSchema = z.enum(["active", "inactive", "suspended"]);
```

#### List Query Command

```typescript
// src/lib/api/commands/queries/hauler/list-haulers.query.ts
import { z } from "zod";
import { createSelectSchema } from "drizzle-zod";
import { and, desc, eq, ilike, count } from "drizzle-orm";
import { haulers, haulerStatusSchema } from "@/lib/db/schema/haulers";

export const listHaulersQueryInputSchema = z
  .object({
    page: z.number().int().min(1).default(1),
    pageSize: z.number().int().min(1).max(100).default(25),
    search: z.string().optional(),
    status: haulerStatusSchema.optional(),
    sortBy: z.enum(["code", "name", "createdAt"]).default("createdAt"),
    sortOrder: z.enum(["asc", "desc"]).default("desc"),
  })
  .strict();

const haulerSelectSchema = createSelectSchema(haulers).pick({
  id: true,
  code: true,
  name: true,
  legalName: true,
  status: true,
  createdAt: true,
  updatedAt: true,
});

export const listHaulersQueryOutputSchema = z
  .object({
    items: z.array(haulerSelectSchema),
    page: z.number().int().min(1),
    pageSize: z.number().int().min(1),
    totalCount: z.number().int().nonnegative(),
    totalPages: z.number().int().nonnegative(),
  })
  .strict();

export type ListHaulersQueryInput = z.infer<typeof listHaulersQueryInputSchema>;
export type ListHaulersQueryOutput = z.infer<
  typeof listHaulersQueryOutputSchema
>;

export async function listHaulersQuery(
  { db }: QueryContext,
  input: ListHaulersQueryInput,
): Promise<ListHaulersQueryOutput> {
  const { page, pageSize, search, status, sortBy, sortOrder } = input;

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

  const sortColumn = haulers[sortBy];
  const orderCondition =
    sortOrder === "asc" ? asc(sortColumn) : desc(sortColumn);

  const items = await db
    .select({
      id: haulers.id,
      code: haulers.code,
      name: haulers.name,
      legalName: haulers.legalName,
      status: haulers.status,
      createdAt: haulers.createdAt,
      updatedAt: haulers.updatedAt,
    })
    .from(haulers)
    .where(whereClause)
    .orderBy(orderCondition)
    .limit(pageSize)
    .offset((page - 1) * pageSize);

  const [totalResult] = await db
    .select({ count: count() })
    .from(haulers)
    .where(whereClause);
  const totalCount = Number(totalResult?.count ?? 0);
  const totalPages = totalCount === 0 ? 0 : Math.ceil(totalCount / pageSize);

  return { items, page, pageSize, totalCount, totalPages };
}
```

#### Create Mutation Command

```typescript
// src/lib/api/commands/mutations/hauler/create-hauler.mutation.ts
import { z } from "zod";
import { ConflictError } from "@/lib/errors/AppError";
import { haulers } from "@/lib/db/schema/haulers";

export const createHaulerMutationInputSchema = z
  .object({
    code: z.string().min(1).max(20),
    name: z.string().min(1).max(100),
    legalName: z.string().min(1).max(200),
  })
  .strict();

export const createHaulerMutationOutputSchema = z
  .object({
    id: z.string().uuid(),
    code: z.string(),
    name: z.string(),
    legalName: z.string(),
    status: z.string(),
    createdAt: z.string(),
  })
  .strict();

export type CreateHaulerMutationInput = z.infer<
  typeof createHaulerMutationInputSchema
>;
export type CreateHaulerMutationOutput = z.infer<
  typeof createHaulerMutationOutputSchema
>;

export async function createHaulerMutation(
  ctx: MutationContext,
  input: CreateHaulerMutationInput,
): Promise<CreateHaulerMutationOutput> {
  const { db, userId } = ctx;
  const { code, name, legalName } = input;

  return await db.transaction(async (tx) => {
    // Check for duplicate code
    const [existing] = await tx
      .select({ id: haulers.id })
      .from(haulers)
      .where(eq(haulers.code, code))
      .limit(1);

    if (existing)
      throw new ConflictError(`Hauler with code ${code} already exists`);

    // Create hauler
    const now = new Date();
    const [created] = await tx
      .insert(haulers)
      .values({
        code,
        name,
        legalName,
        status: "active",
        createdAt: now,
        updatedAt: now,
        createdBy: userId,
        updatedBy: userId,
      })
      .returning({
        id: haulers.id,
        code: haulers.code,
        name: haulers.name,
        legalName: haulers.legalName,
        status: haulers.status,
        createdAt: haulers.createdAt,
      });

    return {
      id: created.id,
      code: created.code,
      name: created.name,
      legalName: created.legalName,
      status: created.status,
      createdAt: created.createdAt.toISOString(),
    };
  });
}
```

#### Router

```typescript
// src/lib/api/routers/haulers.router.ts
import { createTRPCRouter, rbacProcedure } from "../trpc";
import {
  listHaulersQueryInputSchema,
  listHaulersQueryOutputSchema,
  listHaulersQuery,
} from "../commands/queries/hauler/list-haulers.query";
import {
  createHaulerMutationInputSchema,
  createHaulerMutationOutputSchema,
  createHaulerMutation,
} from "../commands/mutations/hauler/create-hauler.mutation";
import {
  handleMasterDataError,
  assertMasterDataPermission,
} from "./_helpers/master-data";
import { PERMISSIONS } from "@/lib/auth/permissions";

const requireHaulerViewPermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.HAULER_VIEW,
    "Missing hauler view permission",
  );
};

const requireHaulerManagePermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.HAULER_MANAGE,
    "Missing hauler manage permission",
  );
};

export const haulersRouter = createTRPCRouter({
  list: rbacProcedure
    .input(listHaulersQueryInputSchema)
    .output(listHaulersQueryOutputSchema)
    .query(async ({ ctx, input }) => {
      try {
        requireHaulerViewPermission(ctx);
        return await listHaulersQuery({ db: ctx.db }, input);
      } catch (err) {
        return handleMasterDataError(err, {
          ctx,
          entity: "hauler",
          action: "list",
        });
      }
    }),

  create: rbacProcedure
    .input(createHaulerMutationInputSchema)
    .output(createHaulerMutationOutputSchema)
    .mutation(async ({ ctx, input }) => {
      try {
        requireHaulerManagePermission(ctx);
        return await createHaulerMutation(ctx, input);
      } catch (err) {
        return handleMasterDataError(err, {
          ctx,
          entity: "hauler",
          action: "create",
          payload: input,
        });
      }
    }),
});
```

#### RSC Usage

```typescript
// app/[locale]/(auth)/dashboard/master-data/haulers/page.tsx
import { api } from '@/lib/api/server'
import { HaulersListClient } from '@/components/features/master-data/HaulersListClient'

export default async function HaulersPage() {
  const caller = await api()
  const haulers = await caller.haulers.list({ page: 1, pageSize: 25 })

  return <HaulersListClient initialData={haulers} />
}
```

#### Client Component

```typescript
// src/components/features/master-data/HaulersListClient.tsx
'use client'
import { api } from '@/lib/api/react'
import type { RouterOutputs } from '@/lib/api/root'

export function HaulersListClient({
  initialData,
}: {
  initialData: RouterOutputs['haulers']['list']
}) {
  const { data } = api.haulers.list.useQuery(
    { page: 1, pageSize: 25 },
    { initialData }
  )

  return (
    <div>
      <h1>Haulers ({data.totalCount})</h1>
      <ul>
        {data.items.map(hauler => (
          <li key={hauler.id}>
            {hauler.code} - {hauler.name}
          </li>
        ))}
      </ul>
    </div>
  )
}
```

## Example 2: List Query with Pagination

### Complete list query implementation with search, filters, and sorting.

```typescript
// Command
export const listStacksQueryInputSchema = z.object({
  page: z.number().int().min(1).default(1),
  pageSize: z.number().int().min(1).max(100).default(25),
  search: z.string().optional(),
  status: stackStatusSchema.optional(),
  warehouseId: z.string().uuid().optional(),
  sortBy: z.enum(['code', 'createdAt', 'closedAt']).default('createdAt'),
  sortOrder: z.enum(['asc', 'desc']).default('desc'),
}).strict()

export const listStacksQueryOutputSchema = z.object({
  items: z.array(stackSelectSchema),
  page: z.number().int().min(1),
  pageSize: z.number().int().min(1),
  totalCount: z.number().int().nonnegative(),
  totalPages: z.number().int().nonnegative(),
}).strict()

export async function listStacksQuery(
  { db }: QueryContext,
  input: ListStacksQueryInput,
): Promise<ListStacksQueryOutput> {
  const { page, pageSize, search, status, warehouseId, sortBy, sortOrder } = input

  const whereConditions = []

  if (search) {
    whereConditions.push(
      or(
        ilike(stacks.code, `%${search}%`),
        ilike(stacks.description, `%${search}%`),
      )
    )
  }

  if (status) whereConditions.push(eq(stacks.status, status))
  if (warehouseId) whereConditions.push(eq(stacks.warehouseId, warehouseId))

  const whereClause = whereConditions.length > 0 ? and(...whereConditions) : undefined

  const sortColumn = stacks[sortBy]
  const orderCondition = sortOrder === 'asc' ? asc(sortColumn) : desc(sortColumn)

  const items = await db
    .select({
      id: stacks.id,
      code: stacks.code,
      status: stacks.status,
      warehouseId: stacks.warehouseId,
      createdAt: stacks.createdAt,
      closedAt: stacks.closedAt,
    })
    .from(stacks)
    .where(whereClause)
    .orderBy(orderCondition)
    .limit(pageSize)
    .offset((page - 1) * pageSize)

  const [totalResult] = await db.select({ count: count() }).from(stacks).where(whereClause)
  const totalCount = Number(totalResult?.count ?? 0)
  const totalPages = totalCount === 0 ? 0 : Math.ceil(totalCount / pageSize)

  return { items, page, pageSize, totalCount, totalPages }
}

// Router
export const stacksRouter = createTRPCRouter({
  list: rbacProcedure
    .input(listStacksQueryInputSchema)
    .output(listStacksQueryOutputSchema)
    .query(async ({ ctx, input }) => {
      try {
        requireStackViewPermission(ctx)
        return await listStacksQuery({ db: ctx.db }, input)
      } catch (err) {
        return handleStackError(err, { ctx, action: 'list' })
      }
    }),
})

// Client with filters
'use client'
export function StacksListClient() {
  const [page, setPage] = useState(1)
  const [search, setSearch] = useState('')
  const [status, setStatus] = useState<StackStatus | undefined>()

  const { data, isLoading } = api.stacks.list.useQuery({
    page,
    pageSize: 25,
    search: search || undefined,
    status,
    sortBy: 'createdAt',
    sortOrder: 'desc',
  })

  return (
    <div>
      <input value={search} onChange={(e) => setSearch(e.target.value)} placeholder="Search..." />
      <select value={status} onChange={(e) => setStatus(e.target.value as StackStatus)}>
        <option value="">All</option>
        <option value="open">Open</option>
        <option value="closed">Closed</option>
      </select>

      {isLoading && <div>Loading...</div>}

      {data && (
        <>
          <ul>
            {data.items.map(stack => (
              <li key={stack.id}>{stack.code} - {stack.status}</li>
            ))}
          </ul>
          <div>
            Page {data.page} of {data.totalPages} ({data.totalCount} total)
          </div>
          <button onClick={() => setPage(p => Math.max(1, p - 1))}>Previous</button>
          <button onClick={() => setPage(p => Math.min(data.totalPages, p + 1))}>Next</button>
        </>
      )}
    </div>
  )
}
```

## Example 3: Protected Mutation with RBAC

### Mutation requiring specific permission with business logic validation.

```typescript
// Domain error
export class InvalidStackStatusError extends ValidationError {
  constructor(currentStatus: string, requiredStatus: string) {
    super(`Stack status is '${currentStatus}' but must be '${requiredStatus}' for this operation`)
  }
}

// Command
export const certifyStackMutationInputSchema = z.object({
  stackId: z.string().uuid(),
  override: z.object({
    reason: z.string().min(10).max(500),
  }).optional(),
}).strict()

export const certifyStackMutationOutputSchema = z.object({
  stackId: z.string().uuid(),
  status: z.literal('certified'),
  certifiedAt: z.string(),
}).strict()

export async function certifyStackMutation(
  ctx: MutationContext,
  input: CertifyStackMutationInput,
): Promise<CertifyStackMutationOutput> {
  const { db, userId } = ctx
  const { stackId, override } = input

  return await db.transaction(async (tx) => {
    const [currentStack] = await tx
      .select({
        id: stacks.id,
        status: stacks.status,
        verifiedBy: stacks.verifiedBy,
      })
      .from(stacks)
      .where(eq(stacks.id, stackId))
      .limit(1)

    if (!currentStack) throw new NotFoundError('Stack')

    if (currentStack.status !== 'verified') {
      throw new InvalidStackStatusError(currentStack.status, 'verified')
    }

    if (currentStack.verifiedBy === userId) {
      throw new SeparationOfDutiesViolationError('certify', 'verify', userId)
    }

    const now = new Date()
    const [certifiedStack] = await tx
      .update(stacks)
      .set({
        status: 'certified',
        certifiedAt: now,
        certifiedBy: userId,
        updatedAt: now,
        updatedBy: userId,
      })
      .where(eq(stacks.id, stackId))
      .returning({ id: stacks.id, status: stacks.status, certifiedAt: stacks.certifiedAt })

    await tx.insert(stackAudits).values({
      stackId,
      action: 'certified',
      actorUserId: userId,
      occurredAt: now,
      beforeJsonb: { status: 'verified' },
      afterJsonb: { status: 'certified', certifiedAt: now.toISOString() },
    })

    return {
      stackId: certifiedStack.id,
      status: 'certified',
      certifiedAt: certifiedStack.certifiedAt!.toISOString(),
    }
  })
}

// Router
const requireStackCertifyPermission = (ctx: PermissionContext) => {
  assertMasterDataPermission(
    ctx,
    PERMISSIONS.STACK_CERTIFY,
    'You do not have permission to certify stacks'
  )
}

export const stacksRouter = createTRPCRouter({
  certify: rbacProcedure
    .input(certifyStackMutationInputSchema)
    .output(certifyStackMutationOutputSchema)
    .mutation(async ({ ctx, input }) => {
      try {
        requireStackCertifyPermission(ctx)
        return await certifyStackMutation(ctx, input)
      } catch (err) {
        return handleStackError(err, { ctx, action: 'certify', payload: input })
      }
    }),
})

// Client with error handling
'use client'
export function CertifyStackButton({ stackId }: { stackId: string }) {
  const queryClient = useQueryClient()
  const [error, setError] = useState<string | null>(null)

  const certify = api.stacks.certify.useMutation({
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: api.stacks.list.queryKey() })
      queryClient.invalidateQueries({ queryKey: api.stacks.get.queryKey({ id: stackId }) })
      setError(null)
    },
    onError: (err) => {
      if (err.data?.code === 'FORBIDDEN') {
        setError('You do not have permission to certify stacks')
      } else if (err.data?.code === 'BAD_REQUEST') {
        setError(err.message)
      } else {
        setError('An unexpected error occurred')
      }
    }
  })

  return (
    <div>
      <button
        onClick={() => certify.mutate({ stackId })}
        disabled={certify.isPending}
      >
        {certify.isPending ? 'Certifying...' : 'Certify Stack'}
      </button>
      {error && <div className="text-red-500">{error}</div>}
    </div>
  )
}
```

## Example 4: Nested Entity Router

### Router with related entities and nested procedures.

```typescript
// Stacks router with nested sub-routers
export const stacksRouter = createTRPCRouter({
  // Top-level procedures
  list: rbacProcedure.input(...).output(...).query(...),
  get: rbacProcedure.input(...).output(...).query(...),

  // Nested lifecycle procedures
  lifecycle: createTRPCRouter({
    certify: rbacProcedure.input(...).output(...).mutation(...),
    verify: rbacProcedure.input(...).output(...).mutation(...),
    close: rbacProcedure.input(...).output(...).mutation(...),
  }),

  // Nested movement procedures
  movement: createTRPCRouter({
    transfer: rbacProcedure.input(...).output(...).mutation(...),
    merge: rbacProcedure.input(...).output(...).mutation(...),
  }),
})

// Client usage
const certify = api.stacks.lifecycle.certify.useMutation()
const transfer = api.stacks.movement.transfer.useMutation()
```

## Example 5: Error Handling Examples

### Comprehensive error handling patterns.

```typescript
// Domain errors
export class StackNotFoundError extends NotFoundError {
  constructor(id: string) {
    super(`Stack with id ${id} not found`)
  }
}

export class InvalidStackStatusError extends ValidationError {
  constructor(currentStatus: string, requiredStatus: string) {
    super(`Stack status is '${currentStatus}' but must be '${requiredStatus}'`)
  }
}

// Command throws domain errors
export async function certifyStackMutation(...) {
  const [stack] = await db.select(...).where(eq(stacks.id, id)).limit(1)

  if (!stack) throw new StackNotFoundError(id)

  if (stack.status !== 'verified') {
    throw new InvalidStackStatusError(stack.status, 'verified')
  }

  // Continue with business logic
}

// Router maps to TRPC errors
export const stacksRouter = createTRPCRouter({
  certify: rbacProcedure
    .input(CertifyStackInputSchema)
    .output(CertifyStackOutputSchema)
    .mutation(async ({ ctx, input }) => {
      try {
        return await certifyStackMutation(ctx, input)
      } catch (err) {
        // Central error handling
        return handleStackError(err, { ctx, action: 'certify', payload: input })
      }
    }),
})

// Client handles errors
const certify = api.stacks.certify.useMutation({
  onError: (error) => {
    if (error.data?.code === 'NOT_FOUND') {
      toast.error('Stack not found')
    } else if (error.data?.code === 'BAD_REQUEST') {
      toast.error(`Validation error: ${error.message}`)
    } else if (error.data?.code === 'FORBIDDEN') {
      toast.error('You do not have permission to perform this action')
    } else {
      toast.error('An unexpected error occurred')
    }
  }
})
```

## Example 6: Server Caller in RSC

### Complete example of using tRPC server caller in Next.js RSC.

```typescript
// Server component
import { api } from '@/lib/api/server'
import { StackDetailClient } from '@/components/features/stacks/StackDetailClient'

export default async function StackDetailPage({ params }: { params: { id: string } }) {
  const { id } = await params // Next 16 - params are async
  const caller = await api()

  // Multiple parallel calls
  const [stack, logs, lineage] = await Promise.all([
    caller.stacks.get({ id }),
    caller.stacks.listLogs({ stackId: id, page: 1, pageSize: 50 }),
    caller.stacks.getLineage({ stackId: id }),
  ])

  return (
    <div>
      <h1>{stack.code}</h1>
      <StackDetailClient stack={stack} logs={logs} lineage={lineage} />
    </div>
  )
}

// Client component
'use client'
import type { RouterOutputs } from '@/lib/api/root'

export function StackDetailClient({
  stack,
  logs,
  lineage,
}: {
  stack: RouterOutputs['stacks']['get']
  logs: RouterOutputs['stacks']['listLogs']
  lineage: RouterOutputs['stacks']['getLineage']
}) {
  // Use initial data from server
  const { data: stackData } = api.stacks.get.useQuery(
    { id: stack.id },
    { initialData: stack }
  )

  // Client-only mutations
  const certify = api.stacks.certify.useMutation()

  return (
    <div>
      <div>Status: {stackData.status}</div>
      <button onClick={() => certify.mutate({ stackId: stack.id })}>
        Certify
      </button>
    </div>
  )
}
```

## Example 7: Client Component with Mutations

### Full client component with mutation and cache invalidation.

```typescript
'use client'
import { api } from '@/lib/api/react'
import { useQueryClient } from '@tanstack/react-query'
import { useState } from 'react'

export function CreateHaulerForm() {
  const queryClient = useQueryClient()
  const [code, setCode] = useState('')
  const [name, setName] = useState('')
  const [legalName, setLegalName] = useState('')
  const [error, setError] = useState<string | null>(null)

  const create = api.haulers.create.useMutation({
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: api.haulers.list.queryKey() })
      setCode('')
      setName('')
      setLegalName('')
      setError(null)
    },
    onError: (err) => {
      if (err.data?.code === 'CONFLICT') {
        setError(`Hauler with code ${code} already exists`)
      } else {
        setError('Failed to create hauler')
      }
    }
  })

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    create.mutate({ code, name, legalName })
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={code}
        onChange={(e) => setCode(e.target.value)}
        placeholder="Code"
        required
      />
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Name"
        required
      />
      <input
        value={legalName}
        onChange={(e) => setLegalName(e.target.value)}
        placeholder="Legal Name"
        required
      />
      <button type="submit" disabled={create.isPending}>
        {create.isPending ? 'Creating...' : 'Create Hauler'}
      </button>
      {error && <div className="text-red-500">{error}</div>}
      {create.isSuccess && <div className="text-green-500">Hauler created successfully</div>}
    </form>
  )
}
```
