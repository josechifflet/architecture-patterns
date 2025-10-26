# Framework - Architecture Documentation

@../CLAUDE.md

Minimal, repeatable patterns for building full-stack apps with Next.js, tRPC, Drizzle, and shadcn/ui.

## Purpose

Documents minimal, repeatable patterns - one pattern per problem, clear boundaries, type safety first.

## Structure

```
framework/
  overview.md           # Philosophy and quick nav
  principles.md         # Design philosophy
  structure.md          # File organization rules
  naming.md            # Naming conventions
  patterns/            # 22 patterns (RSC, tRPC, forms, etc.)
  recipes/             # Step-by-step guides
  examples/            # Full working examples
  decisions/           # ADRs
```

## Commands

```bash
./scripts/qa-patterns.sh           # Validate patterns
./scripts/check-boundaries.sh      # Boundary enforcement
./scripts/check-command-paths.sh   # Canonical paths
```

## Navigation

**Learning**: `overview.md` → `principles.md` → `patterns/index.md` → `examples/crud-feature.md`

**Implementation**: `recipes/` (add-page, add-endpoint, add-table, add-feature)

## Critical Patterns (Master First)

1. `patterns/server-components.md` - RSC data fetching
2. `patterns/client-components.md` - Interactive UI
3. `patterns/trpc-procedures.md` - Router/command boundary
4. `patterns/queries.md` - List normalization
5. `patterns/mutations.md` - Mutation shape + caching
6. `patterns/error-handling.md` - Central error mapping

## Enforcement

- Scripts: `qa-patterns.sh`, `check-boundaries.sh`
- ESLint: Custom rules in `.eslintrc-custom-rules/`
- Tests: Contract tests validate schemas

## Related

- `../README.md` - Project setup
- `../product/` - Feature specs
- `../.claude/skills/` - Claude guides
