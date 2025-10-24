# Tailwind System (v4)

## When to Use
- Always. Tailwind v4 is the only styling system used.
- Define tokens in CSS via `@theme`; import Tailwind once.

## Structure
```
app/
  globals.css       # `@import "tailwindcss";` + tokens
```

## Complete Example
File: `src/app/globals.css`
```css
@import "tailwindcss";

/* Tokens */
@theme inline {
  --font-sans: ui-sans-serif, system-ui, Inter, sans-serif;
  --color-brand: oklch(0.6 0.2 250);
}

/* Optional: custom utility */
@utility tab-* {
  tab-size: *;
}

:root { color-scheme: light; }
.dark { color-scheme: dark; }
```

Setup (PostCSS):
```json
// package.json
{
  "postcss": { "plugins": { "@tailwindcss/postcss": {} } }
}
```

Usage:
```tsx
// app/layout.tsx
import '@/app/globals.css'
```

## Anti-Patterns
❌ Using `tailwind.config.js` for v4 unless absolutely necessary.
✅ Prefer CSS-first `@theme` tokens and imports.

## Checklist
- [ ] `@import "tailwindcss";` once in `globals.css`
- [ ] Tokens declared with `@theme`
- [ ] Dark mode via `.dark` class (if needed)

## Related Patterns
- See: `shadcn-usage.md`
- See: `responsive-patterns.md`

