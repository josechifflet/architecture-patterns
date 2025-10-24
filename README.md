# Tech Stack Frameworks Vault

This repository is a curated vault of minimal, repeatable documentation frameworks for common web stacks. Each framework is a self‑contained set of principles, patterns, recipes, examples, and decision logs you can copy into projects.

Goals
- Minimal files
- Clean architecture
- Strictly repeated patterns
- Every scenario has a how‑to with examples

How It’s Organized
- One top‑level folder per stack
- Inside each folder, consistent docs structure:
  - 00‑OVERVIEW, 01‑PRINCIPLES, 02‑PROJECT‑STRUCTURE, 03‑NAMING‑CONVENTIONS
  - patterns/, recipes/, examples/, decisions/

Available Frameworks
- nextjs-trpc-drizzle-tailwind-shadcn — Next.js 16 + tRPC v11 + Drizzle ORM + Tailwind v4 + shadcn/ui documented framework
  - Start at: `nextjs-trpc-drizzle-tailwind-shadcn/00-OVERVIEW.md`
  - Pattern index: `nextjs-trpc-drizzle-tailwind-shadcn/patterns/00-INDEX.md`
  - Recipe index: `nextjs-trpc-drizzle-tailwind-shadcn/recipes/00-INDEX.md`
  - Decisions: `nextjs-trpc-drizzle-tailwind-shadcn/decisions/00-INDEX.md`

Conventions Across Frameworks
- Single path to do each task; avoid multiple competing approaches
- Co‑locate code by responsibility; server logic stays in server folders
- Prefer type‑safety, validation at boundaries, and explicit dependencies
- Document version locks and rationale in decisions

How to Add a New Framework
1) Create a new top‑level folder named after the stack, e.g. `react-rq-zod-tailwind`.
2) Add the standard files:
   - `00-OVERVIEW.md`, `01-PRINCIPLES.md`, `02-PROJECT-STRUCTURE.md`, `03-NAMING-CONVENTIONS.md`
   - `patterns/00-INDEX.md` and subfolders (`components/`, `data/`, `styling/`, `routing/` as applicable)
   - `recipes/00-INDEX.md` and task recipes
   - `examples/` with at least one end‑to‑end example
   - `decisions/00-INDEX.md` and an initial version‑locking decision
3) Keep examples minimal and copy‑pasteable.
4) Prefer exact dependency versions and note them in `decisions/`.

Usage Flow (Per Framework)
1) Read the overview: `00-OVERVIEW.md`
2) Follow a recipe: `recipes/`
3) Consult patterns when unsure: `patterns/`
4) See why we chose something: `decisions/`

Notes
- These are docs‑first frameworks; no code generators are required.
- You can selectively copy only the parts you need per project.
