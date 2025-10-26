# shadcn/ui

shadcn/ui provides accessible primitives you own. Install components with the CLI and theme via Tailwind v4 tokens or CSS variables.

## When To Use

- Shared UI building blocks (buttons, inputs, menus)
- Consistent theming across features
- Accessible form and layout primitives

## Basic Structure

```bash
npx shadcn@latest init
npx shadcn@latest add button input field
```

```json
// components.json (Tailwind v4)
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": { "config": "", "css": "src/styles/global.css", "baseColor": "neutral", "cssVariables": true },
  "aliases": { "ui": "@/components/ui", "components": "@/components", "lib": "@/lib", "hooks": "@/hooks", "utils": "@/lib/utils" }
}
```

## Complete Example

```tsx
// app/page.tsx
import { Button } from '@/components/ui/button'

export default function Page() {
  return (
    <div className="space-y-4">
      <Button>Primary</Button>
      <Button variant="secondary">Secondary</Button>
    </div>
  )
}
```

```css
/* src/styles/global.css */
@import "tailwindcss";

@theme inline {
  --color-primary: oklch(0.64 0.19 262);
  --color-primary-foreground: oklch(0.98 0.03 262);
}
```

## Common Mistakes

- Editing generated UI in place without using the CLI to add or re‑add
- Forgetting to configure `components.json` for Tailwind v4
- Mixing feature‑specific UI into `src/components/ui/*` instead of feature folders
- Icon‑only controls without `aria-label`; Radix imports outside `src/components/ui/**`

## How To

1. Initialize shadcn with Tailwind v4 CSS path pointing to `src/styles/global.css`.
2. Add components via CLI (`npx shadcn@latest add button input ...`).
3. Theme with `@theme inline` tokens in global CSS; avoid scattering variables.
4. Use primitives in features; keep shared primitives under `src/components/ui/*`.

## Anti-pattern

- Copy-pasting components outside the CLI flow (drifts from registry)
- Storing feature-specific UI in `src/components/ui/*`
- Omitting accessible labels for icon-only buttons

## Related

- styling.md
- forms.md
