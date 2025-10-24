# Responsive Patterns

## When to Use
- All components should adapt from mobile → desktop

## Structure
- Use Tailwind responsive prefixes: `sm:`, `md:`, `lg:`, `xl:`, `2xl:`
- Prefer container classes and spacing scales

## Complete Example
```tsx
export function Toolbar() {
  return (
    <div className="flex flex-col gap-2 md:flex-row md:items-center md:justify-between">
      <h2 className="text-lg md:text-xl">Title</h2>
      <div className="flex flex-wrap gap-2">
        <button className="px-3 py-1 rounded bg-zinc-900 text-white">Action</button>
        <button className="px-3 py-1 rounded border">Secondary</button>
      </div>
    </div>
  )
}
```

## Anti-Patterns
❌ Hard-coded widths at every breakpoint.
✅ Use flex/grid and utility breakpoints.

## Checklist
- [ ] Works on small screens first
- [ ] Uses Tailwind breakpoints only
- [ ] No custom media queries unless necessary

## Related Patterns
- See: `tailwind-system.md`

