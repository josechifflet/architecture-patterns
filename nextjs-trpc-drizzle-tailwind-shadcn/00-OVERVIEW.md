# Framework Overview

## Philosophy
- One way to do things
- Explicit over implicit
- Co-location of related code
- Type-safety first

## Quick Reference
- Adding a feature: See `framework/recipes/new-feature.md`
- Component patterns: See `framework/patterns/components/`
- Database changes: See `framework/recipes/new-database-table.md`

## Stack Versions
- Next.js: 16.0.0 — App Router, Route Handlers, RSC
- tRPC: 11.6.0 — fetch adapter, createCaller, end-to-end types
- Drizzle ORM: 0.44.7 — typed schema, relations, migrations
- Tailwind CSS: 4.1.16 — CSS-first config, `@theme` tokens
- @tailwindcss/postcss: 4.1.16 — Tailwind v4 PostCSS plugin
- shadcn CLI: 3.5.0 — component scaffolding

Lock these exact versions in `package.json` to avoid subtle breaking changes. Revisit quarterly via a Decision entry.

