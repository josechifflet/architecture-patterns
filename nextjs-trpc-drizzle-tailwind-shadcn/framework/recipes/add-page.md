# Add Page

Create a new App Router page with proper layout, metadata, loading, and error boundaries.

## Prerequisites

- App Router enabled
- Global layout at `app/layout.tsx`

## Steps

1. Create the route folder and page

```tsx
// app/(marketing)/about/page.tsx
import { getTranslations } from 'next-intl/server'
import type { Metadata } from 'next'

// Type-safe metadata generation
export async function generateMetadata(): Promise<Metadata> {
  const t = await getTranslations('Marketing.About')
  return {
    title: t('title'),
    description: t('description'),
  }
}

export default async function Page() {
  const t = await getTranslations('Marketing.About')

  return (
    <section className="prose">
      <h1>{t('title')}</h1>
      <p>{t('subtitle')}</p>
    </section>
  )
}
```

2. Add a section layout if needed

```tsx
// app/(marketing)/layout.tsx
import type { ReactNode } from 'react'

interface MarketingLayoutProps {
  readonly children: ReactNode
}

export default function MarketingLayout({ children }: MarketingLayoutProps) {
  return <div className="container max-w-4xl py-12">{children}</div>
}
```

3. Add loading and error boundaries

```tsx
// app/(marketing)/about/loading.tsx
export default function Loading() {
  return <div>Loading…</div>
}
```

```tsx
// app/(marketing)/about/error.tsx
'use client'

import { useEffect } from 'react'

interface ErrorProps {
  readonly error: Error & { digest?: string }
  readonly reset: () => void
}

export default function Error({ error, reset }: ErrorProps) {
  useEffect(() => {
    // Log error to monitoring service
    console.error('Page error:', error)
  }, [error])

  return (
    <div className="flex flex-col items-center gap-4 py-12">
      <h2 className="text-xl font-semibold">Something went wrong</h2>
      <p className="text-sm text-muted-foreground">{error.message}</p>
      <button
        onClick={reset}
        className="rounded-md bg-primary px-4 py-2 text-sm text-primary-foreground"
      >
        Try again
      </button>
    </div>
  )
}
```

4. Add messages

```json
// src/locales/en.json
{
  "Marketing": {
    "About": { "title": "About", "subtitle": "Our product helps teams ship faster." }
  }
}
```

```json
// src/locales/fr.json
{
  "Marketing": {
    "About": { "title": "À propos", "subtitle": "Notre produit aide les équipes à livrer plus vite." }
  }
}
```

5. Use i18n-aware navigation

```tsx
// example usage inside the page or other components
import { Link } from '@/lib/i18n-navigation'

export function MarketingNav() {
  return <Link href="/pricing">Pricing</Link>
}
```

## Verification

- Navigate to `/about` and confirm metadata shows
- See a loading state on slow networks
- Trigger an error locally and confirm it renders
- Verify localized strings render for different locales

## Related

- patterns/layouts.md
- patterns/server-components.md
- patterns/server-i18n.md
