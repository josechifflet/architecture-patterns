# Routes

Use the App Router for file‑system routes, nested layouts, and streaming. Prefer server defaults and pass data to client components for interactivity.

## When To Use

- New pages and layouts
- Route groups to organize without changing URLs
- Dynamic segments and pre‑rendering

## Basic Structure

```txt
app/
- layout.tsx
- page.tsx
- (marketing)/page.tsx
- (app)/layout.tsx
- (app)/dashboard/page.tsx
- (app)/tasks/[id]/page.tsx
- (app)/tasks/loading.tsx
- (app)/tasks/error.tsx
```

## Complete Example

```tsx
// app/(app)/posts/[id]/page.tsx
export const revalidate = 60

export async function generateStaticParams() {
  const posts = await fetch('https://api.vercel.app/blog').then((r) => r.json())
  return posts.map((p: { id: string }) => ({ id: String(p.id) }))
}

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const post = await fetch(`https://api.vercel.app/blog/${id}`).then((r) => r.json())
  return (
    <main>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </main>
  )
}
```

## Caching & Revalidation

- Static: default `cache: 'force-cache'`
- Dynamic: `cache: 'no-store'` for per-request freshness
- ISR: `next: { revalidate: n }` and/or tag requests with `next: { tags: [...] }`

## Common Mistakes

- Not awaiting `params` or `searchParams` in Server Components in Next.js 15
- Using `pages/` patterns in the App Router
- Fetching in client when server can pre‑render or cache

## Related

- server-components.md
- layouts.md
- next16-proxy.md

## How To

1. Organize sections with route groups `(group)/` without changing URLs.
2. Add dynamic segments `[id]` and await `params` in Server Components.
3. Use ISR via `export const revalidate = n` or `fetch(..., { next: { revalidate: n } })`.
4. Prefer server reads and pass data to client islands for interactivity.
5. Use the Next 16 Proxy for global request orchestration.
6. For navigation, use i18n-aware helpers from `next-intl` via `src/lib/i18n-navigation.ts`.

## Anti-pattern

- Using legacy `pages/` patterns in the App Router
- Not awaiting `params`/`searchParams` in Server Components
- Implementing global auth/i18n in ad-hoc route handlers instead of the proxy
