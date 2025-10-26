# Providers

Move client providers down the tree to keep the root layout server-only. Wrap interactive islands or feature sections with client providers as needed.

## When To Use

- Introducing client-only providers (React Query, theme, tooltips) without making the whole app client
- Scoping i18n client provider to components that use `useTranslations()`
- Reducing client JS by keeping `app/layout.tsx` server-only

## How To

1. Keep `app/layout.tsx` as a Server Component; import global CSS only.
2. Create `*ClientProviders.tsx` under a feature or section when hooks are required.
3. Wrap only the subtree that needs client hooks; prefer multiple small islands over a global provider.
4. For i18n, use `getTranslations()` in RSC and only use `NextIntlClientProvider` inside client islands.

## Example

```tsx
// app/layout.tsx (server-only)
import '@/styles/global.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  )
}
```

```tsx
// src/components/features/tasks/TasksClientProviders.tsx
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useState } from 'react'

export function TasksClientProviders({ children }: { children: React.ReactNode }) {
  const [qc] = useState(() => new QueryClient())
  return <QueryClientProvider client={qc}>{children}</QueryClientProvider>
}
```

```tsx
// app/(app)/tasks/page.tsx (RSC composing a client island)
import { TasksClientProviders } from '@/components/features/tasks/TasksClientProviders'
import TasksClient from '@/components/features/tasks/TasksClient'

export default async function Page() {
  return (
    <TasksClientProviders>
      <TasksClient />
    </TasksClientProviders>
  )
}
```

```tsx
// src/components/providers/MarketingProviders.tsx — example i18n provider island
'use client'
import { NextIntlClientProvider, useMessages, useLocale } from 'next-intl'

export function MarketingProviders({ children }: { children: React.ReactNode }) {
  const locale = useLocale()
  const messages = useMessages()
  return (
    <NextIntlClientProvider locale={locale} messages={messages}>
      {children}
    </NextIntlClientProvider>
  )
}
```

## Anti-pattern

- Wrapping the entire app tree in a client provider (turns everything client)
- Using `NextIntlClientProvider` at the root instead of server-first `getTranslations()`
- Mixing provider creation and heavy data fetching inside the same component

## Related

- server-components.md
- rsc-prefetch-hydration.md
- server-i18n.md
