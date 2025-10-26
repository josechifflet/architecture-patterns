# tRPC v11 Skill for Atlas

Complete tRPC implementation patterns for the Atlas framework.

## Structure

```
trpc/
├── SKILL.md                   # Main skill document (491 lines)
│   ├── Overview
│   ├── Core Architecture
│   ├── Procedures and Middleware
│   ├── Implementing Procedures
│   ├── Error Handling
│   ├── Client Usage
│   ├── Type Inference
│   ├── Common Mistakes
│   └── Quick Templates
│
└── references/
    ├── patterns.md            # Pattern catalog (643 lines)
    │   ├── Router Structure
    │   ├── Procedure Types
    │   ├── Middleware Patterns
    │   ├── RBAC Implementation
    │   ├── Input/Output Validation
    │   ├── Error Mapping
    │   ├── Server Caller Usage
    │   └── Client Hooks
    │
    ├── recipes.md             # Step-by-step guides (722 lines)
    │   ├── Creating a New Router
    │   ├── Adding Protected Procedures
    │   ├── Implementing List Queries
    │   ├── Mutations with Cache Invalidation
    │   ├── Error Handling
    │   ├── Nested Router Composition
    │   └── Testing Procedures
    │
    └── examples.md            # Complete implementations (783 lines)
        ├── Complete CRUD Router
        ├── List Query with Pagination
        ├── Protected Mutation with RBAC
        ├── Nested Entity Router
        ├── Error Handling Examples
        ├── Server Caller in RSC
        └── Client Component with Mutations
```

## Key Features

### Architecture Principles

1. **Thin Routers** - Validate, enforce RBAC, delegate, map errors
2. **RBAC at Boundary** - Permission checks ONLY in routers via `rbacProcedure`
3. **Central Error Mapping** - NO inline `new TRPCError()` - use `mapAppErrorToTRPC`
4. **Strict Validation** - Always `.input(Schema.strict())` to reject extra properties
5. **Command Delegation** - Business logic lives in commands, not routers

### Command-Query Separation

**Commands** (pure business logic):

- Export Zod input/output schemas
- Accept typed input (already validated by router)
- Throw domain errors (from `src/lib/errors/`)
- Return typed domain data
- NO parsing, NO RBAC, NO TRPCError

**Routers** (thin orchestrators):

- Validate with `.input(Schema.strict())`
- Enforce RBAC with `rbacProcedure`
- Delegate to command
- Map domain errors to TRPC errors

## Coverage

### Procedures

- publicProcedure (no auth)
- rbacProcedure (protected with auth + RBAC)
- Query procedures (`.query()`)
- Mutation procedures (`.mutation()`)

### Patterns

- List queries with pagination
- Single entity queries
- Create/update/delete mutations
- Nested router composition
- Error handling and mapping
- Server caller in RSC
- React Query hooks
- Cache invalidation

### Real-World Examples

- Complete CRUD router (haulers)
- List with search/filters/sort
- Protected mutation with SoD
- Nested procedures
- Comprehensive error handling
- RSC with server caller
- Client forms with mutations

## Usage

Reference this skill when:

- Creating new tRPC procedures
- Implementing RBAC enforcement
- Setting up error handling
- Working with type inference
- Using server caller in RSC
- Setting up client hooks
- Debugging tRPC issues

## Validation

Before committing tRPC changes:

```bash
pnpm check:types                    # TypeScript validation
pnpm lint                           # ESLint checks
./scripts/qa-patterns.sh            # Pattern enforcement
./scripts/check-boundaries.sh       # Boundary validation
```

## Related

- `framework/patterns/trpc-procedures.md` - Framework pattern docs
- `framework/patterns/queries.md` - List query normalization
- `framework/patterns/mutations.md` - Mutation patterns
- `framework/patterns/error-handling.md` - Error mapping
- `src/lib/api/` - Codebase implementation

## Statistics

- **Total Lines**: 2,639
- **SKILL.md**: 491 lines (overview + quick reference)
- **patterns.md**: 643 lines (comprehensive patterns)
- **recipes.md**: 722 lines (step-by-step guides)
- **examples.md**: 783 lines (full working code)
