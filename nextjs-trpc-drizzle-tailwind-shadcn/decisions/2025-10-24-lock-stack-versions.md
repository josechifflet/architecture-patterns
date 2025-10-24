# Decision: Lock Stack Versions (2025-10-24)

## Context
Frequent minor releases across Next.js, tRPC, Drizzle, and Tailwind can introduce subtle breaking changes or new defaults. To ensure repeatability and minimal files, we lock exact versions and upgrade intentionally.

## Decision
Pin the following versions in `package.json`:
- `next`: `16.0.0`
- `@trpc/server`: `11.6.0`
- `drizzle-orm`: `0.44.7`
- `tailwindcss`: `4.1.16`
- `@tailwindcss/postcss`: `4.1.16`
- `shadcn`: `3.5.0`

## Rationale
- Stabilize behavior across environments
- Keep docs and examples aligned with a known baseline
- Enable fast onboarding with fewer “works on my machine” issues

## Alternatives Considered
1. Use caret (^) ranges — rejected: non-deterministic behavior and hidden changes.
2. Use lockfile only — rejected: upgrades can still slip through via transitive ranges.

## Consequences
- Positive: Deterministic builds, consistent dev experience, fewer regressions
- Negative: Requires periodic chore to update versions

## Update Policy
- Review quarterly. Run `npm outdated`, plan upgrades, update docs if needed, and record a new Decision.

