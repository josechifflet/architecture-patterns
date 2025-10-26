# Framework Overview

This framework documents a minimal, repeatable stack for building full‑stack apps with Next.js App Router, tRPC, Drizzle ORM, Tailwind CSS v4, and shadcn/ui. It prioritizes clear boundaries, few files, and consistent patterns you can copy.

## Philosophy

- Minimal files
- One pattern per problem
- Repetition over abstraction
- Type safety first

## What You Get

- Server‑first routing with the Next.js App Router
- End‑to‑end types with tRPC v11 and Zod
- Typed SQL over PostgreSQL using Drizzle
- Tailwind v4 utilities with a CSS‑first setup
- Accessible UI primitives via shadcn/ui

## Stack Versions

- Next.js: 16 (App Router, React 19)
- tRPC: v11 with TanStack Query v5
- Drizzle ORM: latest v2025 docs features (pg schema, relations, CTEs)
- Tailwind CSS: v4.1 (new @import, @theme, @utility)
- shadcn/ui: latest CLI and components

Use these as baselines. Patch versions may change; follow linked references in patterns and recipes.

## Quick Navigation

- Start here: principles.md
- Project layout: structure.md
- Naming rules: naming.md
- Patterns: patterns/index.md
- Step‑by‑step: recipes/index.md
- Examples: examples/crud-feature.md
- Decisions: decisions/index.md
- i18n and tests: see docs/patterns/i18n.md and docs/patterns/tests-*.md

## Scope and Boundaries

- Server Components fetch via a server caller to tRPC, not by importing the DB or commands in UI
- Routers are thin, validate input/output, and delegate to commands
- UI stays props‑only; interactive containers end with Client suffix

Documented exceptions: webhooks may import commands/DB with signature verification and idempotency when recorded.

Follow these rules to keep repetition high and complexity low.

For request orchestration, use the Next 16 Proxy API [see patterns/next16-proxy.md]. Keep `app/layout.tsx` server-only and push client providers down the tree [see patterns/providers.md].
