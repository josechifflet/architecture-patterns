# Layout Patterns

## When to Use
- Shared chrome (nav, footer) across routes
- Inject global styles, fonts, and providers

## Structure
```
app/
  layout.tsx        # RootLayout wraps all routes
  page.tsx          # Home
  (marketing)/...   # Optional segmented layouts
```

## Complete Example
File: `src/app/layout.tsx`
```tsx
import '@/app/globals.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="bg-white text-gray-900 dark:bg-zinc-950 dark:text-white">
        <header className="border-b">
          <div className="container mx-auto p-4">App</div>
        </header>
        <main className="container mx-auto p-4">{children}</main>
      </body>
    </html>
  )
}
```

## Anti-Patterns
❌ Heavy data fetching in the RootLayout.
✅ Keep RootLayout light, fetch per-route as needed.

## Checklist
- [ ] One root layout
- [ ] Globals imported once (`globals.css`)
- [ ] Minimal logic, no per-route concerns

## Related Patterns
- See: `../routing/app-router.md`

