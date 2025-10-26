# Tailwind CSS v4 Skill

Comprehensive guide for styling Next.js applications with Tailwind CSS v4 using CSS-first configuration.

## Overview

This skill provides complete patterns and examples for Tailwind CSS v4, emphasizing:

- **V4 CSS-First Configuration**: @theme, @import, @utility directives
- **8px Grid System**: Semantic spacing tokens aligned to 8px grid
- **Mobile-First Responsive**: Progressive breakpoint patterns
- **Accessibility First**: Focus states, contrast, WCAG compliance
- **Atlas Conventions**: Custom separators, utilities, dark mode

## Structure

```
tailwindcss/
  SKILL.md                    # Main guide (468 lines)
  references/
    patterns.md               # Detailed implementation patterns (775 lines)
    recipes.md                # Common use case solutions (698 lines)
    examples.md               # Complete working examples (916 lines)
```

## What's Included

### SKILL.md (Main Guide)

- When to use this skill
- Core principles and overview
- Quick reference for common patterns
- V4 specific features and migration
- Responsive design fundamentals
- Accessibility best practices
- Common mistakes and how to avoid them
- Resources and next steps

### references/patterns.md

- Tailwind v4 CSS configuration
- Design tokens and theme customization
- Layout patterns (flex, grid, container)
- Responsive utilities and breakpoints
- Typography scale and font system
- Color system with semantic tokens
- Spacing and sizing (8px grid)
- Custom utilities implementation
- Dark mode patterns
- Accessibility patterns

### references/recipes.md

- Creating custom design tokens
- Responsive layout recipes
- Form styling patterns
- Card and panel layouts
- Navigation patterns
- Button variants
- Loading states
- Error states

### references/examples.md

- Complete dashboard layout with sidebar
- Responsive data table with sorting/filtering
- Multi-step form with validation
- Navigation with active states
- Modal with custom backdrop

## Usage

1. **Start with SKILL.md** for overview and when to use patterns
2. **Reference patterns.md** for detailed implementation guidance
3. **Check recipes.md** for common use cases
4. **Study examples.md** for complete implementations

## Key Features

**V4 Modern Syntax**:

- CSS-first configuration in global.css
- @theme for design tokens
- @utility for custom utilities
- No more tailwind.config.js theme

**8px Grid System**:

- Semantic spacing tokens (xs, sm, md, lg, xl, 2xl, 3xl, 4xl)
- Aligned to 8px base grid
- Consistent spacing across components

**Accessibility**:

- WCAG AA compliant color contrast
- 44px minimum touch targets
- Focus ring utilities
- Screen reader support
- Reduced motion respect

**Dark Mode**:

- Class-based dark mode
- Custom variant configuration
- Semantic color tokens
- Automatic theme switching

## Atlas-Specific Conventions

- Custom separators (dotted after headers, dashed before footers)
- Focus ring utilities (focus-ring, focus-ring-primary)
- Skip link utility for keyboard navigation
- High contrast mode support
- Print styles

## Line Counts

- SKILL.md: 468 lines (under 500 ✓)
- patterns.md: 775 lines
- recipes.md: 698 lines
- examples.md: 916 lines
- Total references: 2,389 lines

## Related Atlas Files

- `/src/styles/global.css` - Main stylesheet
- `/framework/patterns/styling.md` - Styling patterns
- `/framework/patterns/shadcn.md` - Component integration
- `/src/components/ui/button.tsx` - Example implementation

## External Resources

- [Tailwind CSS v4 Docs](https://tailwindcss.com/docs)
- [CSS-First Configuration](https://tailwindcss.com/docs/v4-beta)
- [Migration Guide](https://tailwindcss.com/docs/upgrade-guide)
