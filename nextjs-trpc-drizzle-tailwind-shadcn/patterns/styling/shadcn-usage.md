# shadcn/ui Usage

## When to Use
- Add accessible, composable UI primitives quickly
- Copy-in components you own and can edit

## Structure
```
components/
  ui/
    button.tsx
    input.tsx
components.json        # shadcn config
```

## Complete Example
Initialize and add a component:
```bash
npx shadcn@latest init
npx shadcn@latest add button input label
```

Tailwind v4 notes:
- Set `components.json` → `tailwind.config` empty (v4 discovers automatically)
- Ensure `components.json` → `tailwind.css` points to `src/app/globals.css`
- Prefer CSS variables ON (`cssVariables: true`) for themeable tokens

Example `components.json` (excerpt):
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/app/globals.css",
    "cssVariables": true
  }
}
```

Theming with Tailwind v4 tokens:
```css
/* In globals.css */
:root {
  --background: white;
  --foreground: oklch(0.2 0 0);
}
.dark { /* dark tokens */ }

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
}
```

## Anti-Patterns
❌ Treat components as a black box.
✅ Edit components freely — you own the code.

## Checklist
- [ ] CLI initialized and components added via `add`
- [ ] `globals.css` imported in layout
- [ ] Tokens set for brand look

## Related Patterns
- See: `tailwind-system.md`
- See: `../components/form-patterns.md`

