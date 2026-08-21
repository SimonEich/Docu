# Next.js Cheatsheet – Professionelle Softwareentwicklung

Bezieht sich auf Next.js mit App Router (Next.js 14/15.x). Web-Suche war in dieser Session nicht verfügbar; prüfe bei sicherheitsrelevanten oder versionsspezifischen Details zusätzlich die offizielle Doku unter nextjs.org/docs, da sich Next.js schnell weiterentwickelt.

## 1. Projektstruktur

Eine saubere Struktur ist die Basis für Wartbarkeit im Team.

```
src/
  app/                    # Routing (App Router) – Pages, Layouts, Route Handler
    (marketing)/          # Route Groups für Layout-Trennung ohne URL-Segment
    api/                  # Route Handler (app/api/.../route.ts)
    layout.tsx
    page.tsx
    error.tsx             # Error Boundaries pro Segment
    loading.tsx           # Suspense-Fallbacks pro Segment
    not-found.tsx
  components/
    ui/                   # Reine Präsentationskomponenten (z. B. shadcn/ui)
    features/             # Feature-spezifische Komponenten
  lib/                    # Utilities, API-Clients, Helper-Funktionen
  hooks/                  # Custom React Hooks
  server/                 # Server-only Code (DB-Zugriff, Auth, Actions)
  types/                  # Geteilte TypeScript-Typen
  styles/
middleware.ts
next.config.ts
.env.local
```

Faustregel: Server-only Code (DB-Queries, Secrets) strikt von Client-Code trennen, z. B. mit dem Paket `server-only` importieren, damit versehentliche Client-Imports einen Build-Fehler auslösen.

## 2. Server vs. Client Components

- Standardmäßig sind alle Komponenten im `app/`-Verzeichnis **Server Components** – kein Client-JS, direkter Datenbankzugriff möglich, keine Hooks wie `useState`/`useEffect`.
- `"use client"` nur dort setzen, wo Interaktivität, Browser-APIs oder React-Hooks nötig sind – so klein wie möglich halten ("Client-Inseln").
- Composition-Pattern: Server Components als Children an Client Components übergeben (`children`-Prop), statt Server-Logik in Client-Komponenten zu importieren.
- Datenfetching möglichst in Server Components erledigen (`async function Page()`), nicht per `useEffect` im Client nachladen.

## 3. Data Fetching & Caching

- `fetch()` in Server Components wird von Next.js automatisch dedupliziert und gecacht.
- Cache-Strategien explizit steuern:
  - `fetch(url, { cache: 'force-cache' })` – statisch, Standard für die meisten Fälle
  - `fetch(url, { cache: 'no-store' })` – immer frisch (z. B. personalisierte Daten)
  - `fetch(url, { next: { revalidate: 60 } })` – Incremental Static Regeneration (ISR) alle 60 Sekunden
  - `fetch(url, { next: { tags: ['posts'] } })` + `revalidateTag('posts')` – gezielte Invalidierung nach Mutationen
- Parallel statt sequenziell fetchen: mehrere `fetch`-Aufrufe/Promises parallel starten und erst mit `Promise.all` awaiten, um Wasserfälle zu vermeiden.
- `React.cache()` nutzen, um Datenbank-Queries innerhalb eines Requests zu deduplizieren.

## 4. Server Actions & Mutationen

```tsx
// app/actions.ts
'use server'

import { z } from 'zod'
import { revalidatePath } from 'next/cache'

const schema = z.object({ title: z.string().min(1) })

export async function createPost(formData: FormData) {
  const parsed = schema.safeParse({ title: formData.get('title') })
  if (!parsed.success) return { error: parsed.error.flatten() }

  await db.post.create({ data: parsed.data })
  revalidatePath('/posts')
}
```

- Server Actions immer serverseitig validieren (z. B. mit Zod) – Client-Validierung ist nur UX, kein Sicherheitsmechanismus.
- Nach Mutationen gezielt `revalidatePath()` oder `revalidateTag()` aufrufen, damit Caches konsistent bleiben.
- Für Formulare `useFormStatus` und `useActionState` (früher `useFormState`) für Pending-States und Fehleranzeige nutzen.
- Rate Limiting und Auth-Checks auch in Server Actions durchführen – sie sind öffentlich erreichbare Endpunkte, kein "sicherer" RPC-Mechanismus per se.

## 5. Rendering-Strategien

| Strategie | Wann verwenden |
|---|---|
| Static (SSG) | Inhalte, die selten wechseln (Marketing, Doku) |
| ISR | Inhalte, die periodisch aktualisiert werden (Blog, Produktkatalog) |
| Dynamic (SSR) | Personalisierte/nutzerabhängige Inhalte, Echtzeitdaten |
| Streaming (Suspense) | Lange Ladezeiten einzelner Datenquellen auf sonst schnellen Seiten |

Streaming mit `loading.tsx` bzw. `<Suspense>` einsetzen, damit die Seite schnell interaktiv wird, während langsame Daten nachladen.

## 6. Performance

- `next/image` konsequent statt `<img>` verwenden (automatische Optimierung, Lazy Loading, richtige `sizes`-Angabe für responsive Bilder).
- `next/font` für Web Fonts nutzen – vermeidet Layout Shifts und externe Requests.
- `next/dynamic` für Code-Splitting großer/seltener genutzter Client-Komponenten (z. B. Rich-Text-Editoren, Chart-Bibliotheken).
- Route Groups und Parallel/Intercepting Routes gezielt einsetzen, um Layout-Wiederverwendung zu maximieren und unnötige Re-Renders zu vermeiden.
- Bundle-Analyse regelmäßig mit `@next/bundle-analyzer` prüfen.
- Core Web Vitals im Blick behalten (LCP, INP, CLS) – Lighthouse/PageSpeed Insights in CI einbinden.

## 7. Typsicherheit & Codequalität

- TypeScript strict mode (`"strict": true` in `tsconfig.json`) verpflichtend.
- Zod (oder ähnliches) für Laufzeitvalidierung von Formularen, API-Inputs und Env-Variablen (`@t3-oss/env-nextjs` ist dafür ein bewährtes Paket).
- ESLint mit `eslint-config-next` + Prettier als Formatter, per Pre-Commit-Hook (z. B. Husky + lint-staged) erzwungen.
- Absolute Imports über Path-Aliases (`@/components/...`) statt tiefer Relativpfade.

## 8. Auth & Security

- Middleware (`middleware.ts`) für Route-Protection und Redirects auf Edge-Ebene nutzen, aber sensible Autorisierungs-Checks zusätzlich serverseitig in der jeweiligen Route/Action wiederholen (Middleware ist kein alleiniger Schutz).
- Etablierte Auth-Lösungen bevorzugen (Auth.js/NextAuth, Clerk, Lucia o. ä.) statt Eigenbau.
- Secrets nur server-seitig verwenden; nur explizit mit `NEXT_PUBLIC_`-Präfix versehene Env-Variablen gelangen ins Client-Bundle – niemals Secrets so präfixen.
- Security-Header setzen (CSP, `X-Frame-Options`, `Strict-Transport-Security`) über `next.config.ts` (`headers()`-Funktion) oder Middleware.
- Input-Validierung sowohl clientseitig (UX) als auch serverseitig (Sicherheit) – siehe Server Actions.

## 9. Environment & Konfiguration

```ts
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactStrictMode: true,
  images: { remotePatterns: [{ hostname: 'cdn.example.com' }] },
  experimental: {
    // nur bewusst und mit Doku-Referenz aktivieren
  },
}

export default nextConfig
```

- `.env.local` für lokale Secrets, nie committen (`.gitignore` prüfen).
- Getrennte Env-Sets für Development/Preview/Production, insbesondere bei Vercel-Deployments.
- Env-Variablen zur Build-Zeit validieren (z. B. mit Zod-Schema), damit fehlende Variablen den Build brechen statt zur Laufzeit zu crashen.

## 10. Testing

| Ebene | Tools |
|---|---|
| Unit/Komponenten | Vitest oder Jest + React Testing Library |
| Integration/API | Vitest/Jest gegen Route Handler, MSW für Mocking |
| End-to-End | Playwright (empfohlen) oder Cypress |
| Visuelles Testing | Playwright Screenshots, Chromatic |

- Kritische User-Flows (Login, Checkout, Formulare) mindestens per E2E abdecken.
- CI-Pipeline: Lint → Typecheck → Unit-Tests → Build → E2E (gegen Preview-Deployment).

## 11. Deployment & CI/CD

- Preview-Deployments pro Pull Request (Vercel oder vergleichbar) für Review vor dem Merge.
- Produktions-Build lokal/CI mit `next build` verifizieren, `next start` bzw. Standalone-Output (`output: 'standalone'`) für Docker-Deployments nutzen.
- Health-Checks und Monitoring (z. B. Sentry für Error-Tracking, Vercel Analytics/Speed Insights oder ein anderes APM) von Anfang an einrichten.
- Feature Flags für risikoreiche Änderungen statt langlebiger Feature-Branches.

## 12. Wartbarkeit im Team

- Konventionen dokumentieren (README, CONTRIBUTING.md): Namensschemata für Dateien/Ordner, Commit-Konventionen (z. B. Conventional Commits), Branching-Strategie.
- Design-System/Komponentenbibliothek konsequent wiederverwenden statt Ad-hoc-Styling (z. B. shadcn/ui + Tailwind CSS als verbreitete Kombination).
- Barrierefreiheit (a11y) von Anfang an mitdenken: semantisches HTML, `alt`-Texte, Tastaturbedienbarkeit, Kontraste prüfen (z. B. mit `eslint-plugin-jsx-a11y`).
- Regelmäßige Dependency-Updates (z. B. mit Renovate/Dependabot), insbesondere für Next.js selbst, da sich Best Practices und APIs schnell weiterentwickeln.

## 13. Häufige Fallstricke

- `useEffect` für Datenfetching im Client verwenden, obwohl Server Components das direkt und schneller könnten.
- Zu große `"use client"`-Grenzen, die unnötig viel JavaScript zum Client schicken.
- Caching-Verhalten von `fetch` nicht verstehen und dadurch veraltete oder unerwartet frische Daten ausliefern.
- Secrets versehentlich mit `NEXT_PUBLIC_`-Präfix versehen.
- Fehlende serverseitige Validierung, weil man sich auf Client-seitige Formularvalidierung verlässt.
- `any`-Typen und deaktiviertes TypeScript-strict-mode "für später" – erhöht technische Schulden schnell.

---

*Hinweis: Next.js entwickelt sich sehr schnell weiter. Vor Projektstart empfiehlt es sich, die aktuelle Version und den Upgrade-Guide unter nextjs.org/docs zu prüfen.*
