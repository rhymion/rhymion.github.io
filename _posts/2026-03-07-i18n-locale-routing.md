---
layout: post
title: "Internationalization (i18n) — Locale-Based Routing with Next.js"
---

# Internationalization (i18n) — Locale-Based Routing with Next.js

## Overview

In many cases apps must support multiple locales. We can use **next-intl v4** with **locale-based URL routing** in Next.js.
Every page URL is prefixed by the locale (`/en/booking`, `/ja/booking`), which gives:

- **SEO** — search engines index each locale at its own URL
- **CDN caching** — responses are cacheable per locale without cookie-based variance
- **Shareable links** — the locale is self-contained in the URL, not hidden in a cookie
- **Server-side rendering** — the locale is known at request time without client-side detection

In our case, we try to support the locales `en` (English, default) and `ja` (Japanese).

---

## File Structure

```
i18n/
  routing.ts          ← supported locales + default locale
  request.ts          ← server-side getRequestConfig (loads messages)
  navigation.ts       ← locale-aware Link, useRouter, usePathname, redirect

messages/
  en.json             ← English translation strings
  ja.json             ← Japanese translation strings

proxy.ts              ← Next.js middleware: locale routing + auth token check
app/
  layout.tsx          ← minimal root shell; reads locale via getLocale()
  [locale]/
    layout.tsx        ← locale layout: wraps NextIntlClientProvider + page shell
    @header/page.tsx  ← header with locale switcher (client component)
    @sidebar/page.tsx ← nav sidebar (client component)
    @footer/page.tsx  ← footer
    providers.tsx     ← SessionProvider + SidebarProvider
    login/page.tsx
    register/page.tsx
    [entity]/...      ← all generated entity pages
  api/                ← API routes (no locale prefix — unaffected by i18n)
```

---

## Key Files Explained

### `i18n/routing.ts`

Defines the supported locales and the default. Import this wherever the locale list is needed (e.g. the header locale switcher, `generateStaticParams`).

```ts
import { defineRouting } from 'next-intl/routing';

export const routing = defineRouting({
  locales: ['en', 'ja'],
  defaultLocale: 'en',
});
```

### `i18n/request.ts`

Called by next-intl on every server request. Reads the locale from the URL (set by the middleware) and loads the matching message file.

```ts
import { getRequestConfig } from 'next-intl/server';
import { routing } from './routing';

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;
  if (!locale || !routing.locales.includes(locale as 'en' | 'ja')) {
    locale = routing.defaultLocale;
  }
  return {
    locale,
    messages: (await import(`../messages/${locale}.json`)).default,
  };
});
```

### `i18n/navigation.ts`

Re-exports locale-aware navigation helpers created by next-intl. **Always import `Link`, `useRouter`, `usePathname`, and `redirect` from here** instead of `next/link` or `next/navigation` — the wrappers automatically prepend the current locale to every path.

```ts
import { createNavigation } from 'next-intl/navigation';
import { routing } from './routing';

export const { Link, redirect, usePathname, useRouter, getPathname } =
  createNavigation(routing);
```

### `proxy.ts` (middleware)

The project uses `proxy.ts` as its middleware entry point. 
(While `middleware.ts` had been used, from Next.js 16 `proxy` is recommended instead). It chains two responsibilities:

1. **Locale routing** — next-intl redirects bare paths (`/booking` → `/en/booking`) and sets the locale for the request
2. **Auth protection** — non-public paths require a valid JWT; unauthenticated users are redirected to `/{locale}/login`

```ts
// proxy.ts (simplified)
const intlMiddleware = createIntlMiddleware(routing);
const PUBLIC_PATHS = ['/login', '/register'];

export async function proxy(req: NextRequest) {
  // determine path without locale prefix
  const isPublicPath = PUBLIC_PATHS.some(p => pathnameWithoutLocale === p);
  const intlResponse = intlMiddleware(req);   // handles locale redirect/rewrite

  if (isPublicPath) return intlResponse;

  const token = await getToken({ req, secret: process.env.AUTH_SECRET });
  if (!token) {
    url.pathname = `/${locale}/login`;
    return NextResponse.redirect(url);
  }
  return intlResponse;
}

export const config = {
  matcher: ['/((?!api|_next|_vercel|.*\\..*).*)'],
};
```

> **Do not create `middleware.ts`** — the framework detects both files and throws an error.

---

## Using Translations

### In server components

```ts
import { getTranslations } from 'next-intl/server';

export default async function MyPage() {
  const t = await getTranslations('Nav');
  return <h1>{t('home')}</h1>;
}
```

### In client components

```tsx
"use client";
import { useTranslations } from 'next-intl';

export default function MyComponent() {
  const t = useTranslations('Header');
  return <button>{t('signOut')}</button>;
}
```

> **Note:** `useTranslations` only works inside `NextIntlClientProvider`. The locale layout (`app/[locale]/layout.tsx`) wraps all pages in this provider, so any client component under `[locale]/` can call `useTranslations`.

### In `app/[locale]/layout.tsx`

The locale layout must call `setRequestLocale` (for static rendering support) and fetch messages before rendering:

```tsx
import { NextIntlClientProvider } from 'next-intl';
import { getMessages, setRequestLocale } from 'next-intl/server';

export default async function LocaleLayout({ children, params }) {
  const { locale } = await params;
  setRequestLocale(locale);
  const messages = await getMessages();

  return (
    <NextIntlClientProvider messages={messages}>
      <Providers>...</Providers>
    </NextIntlClientProvider>
  );
}
```

---

## Adding a New Locale

1. Add the locale to `i18n/routing.ts`:
   ```ts
   locales: ['en', 'ja', 'fr'],
   ```

2. Create the message file `messages/fr.json` with all required keys (copy `en.json` as a starting template).

3. Update the `localeLabels` map in `app/[locale]/@header/page.tsx`:
   ```ts
   const localeLabels: Record<string, string> = {
     en: "EN",
     ja: "日本語",
     fr: "FR",
   };
   ```

4. Update `i18n/request.ts` — widen the type guard to include the new locale:
   ```ts
   if (!locale || !routing.locales.includes(locale as 'en' | 'ja' | 'fr')) {
   ```

That's all — next-intl handles the rest automatically.

---

### Switching locale

From a client component, use `useRouter().replace` with the `locale` option:

```tsx
const router = useRouter();     // from @/i18n/navigation
const pathname = usePathname(); // from @/i18n/navigation

router.replace(pathname, { locale: 'ja' });
```

The header already exposes this as labelled buttons (EN / 日本語).

---

## Message File Structure

```json
{
  "Header": {
    "signIn": "Sign In",
    "signOut": "Sign Out",
    "openMenu": "Open menu",
    "closeMenu": "Close menu"
  },
  "Nav": {
    "home": "Home",
    "dbTables": "DB Tables",
    ...
  },
  "Auth": {
    "signInTitle": "Sign in to your account",
    "registerTitle": "Create your account",
    ...
  }
}
```

Add new namespaces (top-level keys) as features grow. Keep key names camelCase and scoped to their UI context.

---

## Testing

### Cypress E2E

All `cy.visit()` calls use the `/en/` prefix (the default locale):

```ts
cy.visit('/en/booking');
cy.visit('/en/booking/new');
cy.visit('/en/');
```

This is baked into the test templates (`utils/scripts/templates-test.ts`). Regenerating tests will produce the correct paths automatically.

### Vitest unit tests

NextIntlClientProvider is also needed in (component) testing, when the target component uses translation. 
Unlike in components, if it is acceptable to hard-code locale in testing, code is slightly simpler.

```tsx
import { render } from '@testing-library/react';
import { NextIntlClientProvider } from 'next-intl';
import messages from '@/messages/en.json';

export function renderWithIntl(ui: React.ReactElement) {
  return render(
    <NextIntlClientProvider locale="en" messages={messages}>
      {ui}
    </NextIntlClientProvider>
  );
}
```

We can now call the target component with `renderWithIntl` as wrapper. 
Of course we can also use mock instead, if i18n itself is not the target of testing.

---

### Additional Resources

- 📚 [Next.js (app router) internationalization documentation](https://nextjs.org/docs/app/guides/internationalization)
- 🔧 [Next.js (pages router) internationalization documentation](https://nextjs.org/docs/pages/guides/internationalization)

---

*Tags: Next.js, i18n, Testing, E2E Testing, Cypress, TypeScript*
