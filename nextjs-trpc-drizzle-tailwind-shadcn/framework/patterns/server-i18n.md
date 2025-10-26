# Server-first i18n

## When to use

Use on server-rendered pages and layouts where translations are needed without introducing client providers. Prefer `getTranslations()` in RSC and `setRequestLocale()` in server layouts. Only introduce `NextIntlClientProvider` inside client islands that truly need `useTranslations()`.

## Structure

- Page RSC: call `getTranslations()` in `page.tsx`.
- Auth layout (server): call `setRequestLocale(locale)` in `layout.tsx`.
- Client components: use `useTranslations()` only inside client boundaries.
 - Routing and navigation: define locales with `next-intl/routing` and use `createNavigation` helpers.
 - Messages loader: export `getRequestConfig` and lazy-load `src/locales/<locale>.json`.

## Example

```ts
// src/app/[locale]/(auth)/dashboard/admin/page.tsx
import { getTranslations } from 'next-intl/server';
import { auth } from '@clerk/nextjs/server';
import { api } from '@/lib/api/server';
import { redirect } from 'next/navigation';

export default async function AdminPage() {
  const t = await getTranslations('Admin');
  const { userId } = await auth();
  if (!userId) redirect('/');
  const caller = await api();
  const currentUser = await caller.users.getCurrentUser({ clerkId: userId });
  // ... render using t('title') etc.
}
```

```ts
// src/app/[locale]/(auth)/layout.tsx
import { setRequestLocale } from 'next-intl/server';

export default async function AuthLayout(props: { params: Promise<{ locale: string }>; children: React.ReactNode }) {
  const { locale } = await props.params;
  setRequestLocale(locale);
  return props.children as React.ReactElement;
}
```

```ts
// src/lib/i18n-routing.ts — central routing config
import { defineRouting } from 'next-intl/routing';
import { AppConfig } from '@/lib/config/app-config';

export const routing = defineRouting({
  locales: AppConfig.locales,
  localePrefix: AppConfig.localePrefix,
  defaultLocale: AppConfig.defaultLocale,
});
```

```ts
// src/lib/i18n-navigation.ts — typed navigation helpers
import { createNavigation } from 'next-intl/navigation';
import { routing } from './i18n-routing';

export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
```

```ts
// src/lib/i18n.ts — messages loader per locale
import { getRequestConfig } from 'next-intl/server';
import { routing } from './i18n-routing';

export default getRequestConfig(async ({ requestLocale }) => {
  type Locale = (typeof routing.locales)[number];
  const isLocale = (v: string): v is Locale => (routing.locales as readonly string[]).includes(v);

  let locale = await requestLocale;
  if (!locale || !isLocale(locale)) {
    locale = routing.defaultLocale;
  }
  return { locale, messages: (await import(`../locales/${locale}.json`)).default };
});
```

## Anti-patterns

- Wrapping the entire app tree in `NextIntlClientProvider`. This makes everything a Client Component and defeats RSC.
- Calling `useTranslations()` in server components. Use `getTranslations()` instead.
- Doing locale detection in multiple places; use the proxy + `setRequestLocale()` consistently.

## How To

1. Use Next 16 Proxy to normalize and guard requests; delegate i18n to `next-intl` middleware using `createMiddleware(routing)`.
2. Declare routing in `src/lib/i18n-routing.ts` and import it in `i18n-navigation.ts` and `i18n.ts`.
3. In Server Components, call `getTranslations()`; in server layouts, call `setRequestLocale()` when using `[locale]` segments.
4. Only add `NextIntlClientProvider` in client islands that use `useTranslations()`; prefer RSC `getTranslations()` for pages/layouts.
5. Keep root layout server-only; avoid global client providers; see `patterns/providers.md`.
