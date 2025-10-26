# Styling

Tailwind CSS v4 uses a CSS‑first setup with `@import`, `@theme`, and `@utility`. Centralize tokens in global CSS and compose utilities in components.

## When To Use

- Project‑wide design tokens and utilities
- Consistent spacing, color, and typography
- Component‑level overrides via utilities

## Basic Structure

```css
/* src/styles/global.css */
@layer theme, base, components, utilities;
@import "tailwindcss";

@theme inline {
  --color-primary: oklch(0.64 0.19 262);
}

@utility container {
  margin-inline: auto;
  padding-inline: 2rem;
}
```

## Complete Example

```tsx
// app/layout.tsx
import '@/styles/global.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="bg-white text-zinc-950 dark:bg-zinc-950 dark:text-white">
        <div className="container max-w-5xl py-8">{children}</div>
      </body>
    </html>
  )
}
```

Use `@utility` to define custom utilities that replace v3 `@layer utilities`.

```css
@utility tab-4 {
  tab-size: 4;
}
```

## Common Mistakes

- Using `@tailwind base/components/utilities` instead of `@import "tailwindcss";`
- Relying on v3 opacity utilities; use `bg-black/50` style modifiers
- Missing the Tailwind v4 PostCSS plugin configuration (`@tailwindcss/postcss`)
- Overriding the built-in `dark:` variant incorrectly; prefer the default or use a data-attribute variant when required

Note: To use an attribute-driven dark mode instead of the default `.dark` class, define a variant:

```css
@custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *));
```

## How To

1. Create `src/styles/global.css` with `@import "tailwindcss";` and `@layer theme, base, components, utilities;`.
2. Define tokens in `@theme inline` and custom utilities in `@utility`.
3. Import the CSS only in `app/layout.tsx`.
4. Use utilities/tokens in components; keep Tailwind config minimal for v4.

## Anti-pattern

- Using legacy `@tailwind base/components/utilities` in v4 projects
- Spreading theme tokens across many files instead of a single global CSS
- Overriding variants without `@custom-variant` when needed

## Related

- shadcn.md
- layouts.md
