# Layouts

Layouts define persistent UI around route content. Use nested layouts and route groups to compose shells, navigation, and theming.

## When To Use

- Shared chrome across pages
- Section‑specific shells (e.g., app vs. marketing)
- Streaming route trees with loading states

## Basic Structure

```tsx
// app/layout.tsx
import '@/styles/global.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

## Complete Example

```tsx
// app/(app)/layout.tsx
export default function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-dvh grid grid-rows-[auto_1fr]">
      <header className="p-4 border-b">App</header>
      <main className="p-6">{children}</main>
    </div>
  )
}
```

Use route groups to scope layouts without affecting the URL.

```txt
app/
- (marketing)/layout.tsx
- (marketing)/page.tsx
- (app)/layout.tsx
- (app)/dashboard/page.tsx
```

## Common Mistakes

- Putting data logic in a layout when it belongs in pages
- Forgetting to colocate `loading.tsx` with the route boundary
- Over‑nesting layouts when a route group is enough

## Related

- routes.md
- server-components.md

## How To

1. Keep the root `app/layout.tsx` server-only; import global CSS.
2. Create section layouts with route groups like `(app)/layout.tsx`.
3. Place `loading.tsx`/`error.tsx` alongside the route boundary that streams.
4. Avoid fetching data in layouts unless it is persistent chrome needed for all children.

## Anti-pattern

- Data-heavy fetching in a layout that belongs in pages
- Over-nesting layouts when a simple route group is enough
- Turning the root layout into a client by accident (providers belong in islands)
