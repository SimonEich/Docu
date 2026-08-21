# Ein großes Next.js-Projekt von Null starten – Schritt für Schritt am Praxisbeispiel

Beispielprojekt: **"TaskFlow"** – ein B2B-SaaS-Projektmanagement-Tool (Teams, Projekte, Tickets, Kommentare, Benachrichtigungen), vergleichbar mit Linear/Jira. Ein realistisches "großes" Projekt: mehrere Domänen, mehrere Rollen, Multi-Tenancy, Echtzeit-Elemente, langfristige Wartung im Team.

Die Reihenfolge unten ist bewusst so gewählt, dass jeder Schritt auf dem vorherigen lauffähig aufbaut – nach jedem Schritt hat man etwas, das man committen und zeigen kann. Das ist wichtiger, als von Anfang an "perfekt" zu strukturieren.

## Schritt 0: Bevor überhaupt Code geschrieben wird

Auch wenn der Plan laut Annahme schon steht, kurz die Dinge festhalten, die die technischen Entscheidungen der nächsten Schritte treiben:

- Wer sind die Nutzer, wie viele Tenants/Organisationen, wie viel Traffic realistisch (beeinflusst DB-Wahl, Caching, ob Multi-Tenancy nötig ist)
- Team-Größe und Erfahrung (beeinflusst Monorepo vs. Single-Repo, wie viel Tooling sich lohnt)
- Deployment-Ziel (Vercel, eigener Server, Docker/Kubernetes) – beeinflusst z. B. `output: 'standalone'`, Edge- vs. Node-Runtime

Für TaskFlow: mehrere Firmenkunden (Multi-Tenant), Team von 4 Entwicklern, Deployment auf Vercel, Start mit Postgres.

## Schritt 1: Repository & Grundgerüst aufsetzen

```bash
npx create-next-app@latest taskflow \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*"
cd taskflow
git init && git add -A && git commit -m "chore: initial next.js scaffold"
```

Direkt danach, bevor irgendeine Fachlogik entsteht:

1. **Monorepo-Entscheidung treffen.** Bei einem "großen" Projekt lohnt sich oft schon früh ein Monorepo (Turborepo oder Nx), auch wenn zunächst nur eine App existiert – weil man absehen kann, dass später ein Admin-Panel, ein Marketing-Site oder ein Worker-Service dazukommen.
   ```bash
   npx create-turbo@latest
   # Struktur: apps/web (die Next.js-App), packages/ui, packages/config, packages/db
   ```
   Für TaskFlow: Turborepo, weil absehbar ist, dass ein separates Admin-Dashboard und später ggf. eine mobile API dazukommen.

2. **Node-Version fixieren** (`.nvmrc` + `"engines"` in `package.json`), damit alle im Team und CI dieselbe Version nutzen.

3. **Linting/Formatting/Commit-Konventionen sofort einrichten**, nicht "später":
   ```bash
   npm i -D prettier eslint-config-prettier husky lint-staged @commitlint/cli @commitlint/config-conventional
   npx husky init
   ```
   Pre-Commit-Hook: `lint-staged` (ESLint + Prettier auf geänderte Dateien), Commit-Message-Hook: Conventional Commits. Das verhindert Stilstreit im Team von Tag 1 an.

4. **CI-Grundgerüst** (GitHub Actions o. ä.) mit den Jobs `install → lint → typecheck → build` – auch wenn es noch fast nichts zu testen gibt. Ein leeres, aber funktionierendes CI-Skelett ist deutlich einfacher zu erweitern als eins, das man erst bei Feature 15 nachrüstet.

**Ergebnis nach Schritt 1:** Ein leeres, aber lauffähiges, sauber konfiguriertes Projekt, das gepusht ist und in CI grün läuft.

## Schritt 2: Fundament-Entscheidungen technisch verankern

Jetzt die zentralen Architekturentscheidungen als Code festmachen, bevor Features entstehen:

- **Datenbank & ORM:** Für TaskFlow Postgres + Prisma (oder Drizzle). Schema-Datei anlegen, aber zunächst nur mit den absolut zentralen Entitäten (siehe Schritt 3).
- **Auth:** Auth.js (NextAuth) oder ein Provider wie Clerk. Für ein B2B-Tool mit Organisationen: Auth-Lösung wählen, die Multi-Tenancy/Organizations unterstützt (Clerk hat das eingebaut; mit Auth.js baut man Organisationen selbst über das Datenmodell).
- **UI-Basis:** Tailwind + shadcn/ui als Komponentenbasis installieren, Design-Tokens (Farben, Radius, Spacing) einmal zentral definieren.
- **Env-Validierung:** `@t3-oss/env-nextjs` einrichten, damit fehlende Umgebungsvariablen den Build sofort brechen statt erst zur Laufzeit.

```bash
npm i @prisma/client zod
npm i -D prisma
npx prisma init --datasource-provider postgresql
```

**Wichtig:** In diesem Schritt noch keine Fachlogik – nur die Infrastruktur, auf der jedes Feature aufsetzt.

## Schritt 3: Das Datenmodell für den Kern-Anwendungsfall

Nicht das gesamte Datenmodell im Voraus perfekt entwerfen – aber den Kern, der das ganze Produkt trägt, sauber modellieren. Bei TaskFlow ist das: Organisation → Projekt → Ticket.

```prisma
// packages/db/schema.prisma
model Organization {
  id        String   @id @default(cuid())
  name      String
  slug      String   @unique
  members   Membership[]
  projects  Project[]
  createdAt DateTime @default(now())
}

model Membership {
  id             String       @id @default(cuid())
  role           Role         @default(MEMBER)
  user           User         @relation(fields: [userId], references: [id])
  userId         String
  organization   Organization @relation(fields: [organizationId], references: [id])
  organizationId String

  @@unique([userId, organizationId])
}

model Project {
  id             String       @id @default(cuid())
  name           String
  organization   Organization @relation(fields: [organizationId], references: [id])
  organizationId String
  tickets        Ticket[]
}

model Ticket {
  id        String   @id @default(cuid())
  title     String
  status    Status   @default(TODO)
  project   Project  @relation(fields: [projectId], references: [id])
  projectId String
  createdAt DateTime @default(now())
}

enum Role { OWNER ADMIN MEMBER }
enum Status { TODO IN_PROGRESS DONE }
```

```bash
npx prisma migrate dev --name init
```

Das Prinzip: **vertikale statt horizontale Entwicklung.** Nicht erst alle Tabellen für alle 20 Features anlegen, sondern die Tabellen, die man für den ersten End-to-End-Anwendungsfall braucht (hier: Org anlegen → Projekt anlegen → Ticket anlegen → Ticket anzeigen).

## Schritt 4: Den ersten vertikalen Feature-Slice bauen

Statt "erst alle Komponenten, dann alle Server Actions, dann alle Seiten" zu bauen, wird **ein komplettes Feature end-to-end** durchgezogen: Ticket-Liste eines Projekts anzeigen und ein Ticket erstellen. Das zwingt dazu, Routing, Datenfetching, Server Actions, UI und Validierung von Anfang an im Zusammenspiel zu sehen, statt es am Ende zusammenzustecken.

Routing-Struktur:

```
src/app/
  (dashboard)/
    layout.tsx                         # gemeinsames Layout (Sidebar, Header)
    orgs/[orgSlug]/
      projects/[projectId]/
        page.tsx                       # Ticket-Liste
        actions.ts                     # Server Actions für dieses Segment
        new-ticket-form.tsx            # Client-Komponente
```

```tsx
// app/(dashboard)/orgs/[orgSlug]/projects/[projectId]/page.tsx
import { db } from '@/server/db'
import { NewTicketForm } from './new-ticket-form'

export default async function ProjectPage({
  params,
}: {
  params: Promise<{ projectId: string }>
}) {
  const { projectId } = await params
  const tickets = await db.ticket.findMany({ where: { projectId } })

  return (
    <div>
      <h1 className="text-xl font-semibold">Tickets</h1>
      <ul>
        {tickets.map((t) => (
          <li key={t.id}>{t.title} — {t.status}</li>
        ))}
      </ul>
      <NewTicketForm projectId={projectId} />
    </div>
  )
}
```

```ts
// actions.ts
'use server'

import { z } from 'zod'
import { db } from '@/server/db'
import { revalidatePath } from 'next/cache'
import { requireMembership } from '@/server/auth'

const schema = z.object({
  title: z.string().min(1).max(200),
  projectId: z.string().cuid(),
})

export async function createTicket(formData: FormData) {
  const parsed = schema.safeParse({
    title: formData.get('title'),
    projectId: formData.get('projectId'),
  })
  if (!parsed.success) return { error: parsed.error.flatten() }

  await requireMembership(parsed.data.projectId) // Autorisierung serverseitig!
  await db.ticket.create({ data: parsed.data })
  revalidatePath(`/orgs/.../projects/${parsed.data.projectId}`)
}
```

```tsx
// new-ticket-form.tsx
'use client'
import { useActionState } from 'react'
import { createTicket } from './actions'

export function NewTicketForm({ projectId }: { projectId: string }) {
  const [state, action, pending] = useActionState(createTicket, null)
  return (
    <form action={action}>
      <input type="hidden" name="projectId" value={projectId} />
      <input name="title" placeholder="Ticket-Titel" />
      <button disabled={pending}>Erstellen</button>
      {state?.error && <p className="text-red-600">Ungültige Eingabe</p>}
    </form>
  )
}
```

Was dieser Schritt bewusst zeigt:

- Server Component lädt Daten direkt (`db.ticket.findMany`) – kein API-Layer nötig für internen Gebrauch.
- Server Action übernimmt Mutation + Validierung + Autorisierung + Cache-Invalidierung an einem Ort.
- Die Client-Komponente ist auf das absolute Minimum reduziert (nur das Formular).
- `requireMembership` kapselt von Anfang an die Multi-Tenancy-Prüfung – bei einem SaaS mit Organisationen ist "gehört diese Ressource zur aktiven Organisation des Nutzers" die häufigste Sicherheitslücke, wenn man sie nicht von Anfang an zentralisiert.

**Ergebnis nach Schritt 4:** Ein einziges, aber vollständiges Feature läuft von der Datenbank bis zur UI. Das ist der Moment, um die bisherigen Muster (Ordnerstruktur, Server-Action-Pattern, Auth-Check) im Team zu reviewen, *bevor* sie zehnmal kopiert werden.

## Schritt 5: Auth & Multi-Tenancy vor der Breite ausbauen

Erst nachdem das Kern-Slice steht, Auth vollständig verdrahten:

- Login/Signup-Flow, Session-Handling
- Middleware (`middleware.ts`), die nicht eingeloggte Nutzer aus `/orgs/*` fernhält
- Organisationswechsel (Nutzer kann in mehreren Organisationen Mitglied sein) im Layout verankern, z. B. per Cookie/Subdomain für die aktive Organisation
- Zentrale Autorisierungs-Helper (`requireMembership`, `requireRole`) in `src/server/auth.ts`, die **jede** Server Action und jede Seite verwendet – nicht pro Feature neu erfinden

Das lohnt sich jetzt und nicht früher, weil man am Kern-Slice aus Schritt 4 schon gesehen hat, wie und wo Autorisierung tatsächlich gebraucht wird.

## Schritt 6: Breite statt Tiefe – weitere Features nach demselben Muster

Ab jetzt wird jedes weitere Feature (Kommentare, Zuweisungen, Benachrichtigungen, Filter, Suche) nach demselben, jetzt etablierten Muster gebaut:

1. Datenmodell erweitern (Prisma-Migration)
2. Route/Segment anlegen
3. Server Component für Read, Server Action für Write
4. Autorisierung über die zentralen Helper
5. UI-Komponenten aus `packages/ui` wiederverwenden, nur projektspezifische Komposition neu bauen

Wichtig für "großes Projekt, viele Entwickler": Features möglichst **vertikal pro Domäne** aufteilen (eine Person baut "Kommentare" komplett von DB bis UI), nicht horizontal ("eine Person baut alle Server Actions, eine andere alle Components") – das reduziert Merge-Konflikte und Kontext-Switching.

## Schritt 7: Tests parallel zum Feature-Ausbau, nicht danach

Sobald 2–3 Features stehen, keine "Test-Nachholphase" einplanen, sondern ab jetzt pro neuem Feature direkt mitliefern:

- Unit-Tests für reine Funktionen (z. B. Validierungs-Schemas, Berechnungen) mit Vitest
- Für Server Actions: Integrationstests, die gegen eine Test-Datenbank laufen (z. B. mit Docker-Postgres in CI)
- Für den kritischsten Flow (Org anlegen → Projekt anlegen → Ticket anlegen → sichtbar für Teammitglied, nicht sichtbar für andere Organisation) einen Playwright-E2E-Test – dieser eine Test schützt am meisten, weil er genau die Multi-Tenancy-Grenze prüft, die am teuersten wäre, wenn sie bricht.

## Schritt 8: Observability & Deployment-Pipeline vervollständigen

Parallel zum Feature-Ausbau, spätestens bevor echte Nutzer draufkommen:

- Fehler-Tracking (z. B. Sentry) einrichten, inklusive Server-Action-Fehlern
- Preview-Deployments pro PR (Vercel) als festen Teil des Review-Prozesses
- Datenbank-Migrationen als eigenen, expliziten CI/CD-Schritt (nicht "läuft schon automatisch mit"), damit Migrationen nie unbeabsichtigt in Produktion laufen
- Basic-Metriken/Logs für Server Actions und API-Routen, damit man bei Problemen nicht blind ist

## Zusammengefasste Reihenfolge

1. Repo, Tooling, CI-Skelett aufsetzen (leer, aber sauber)
2. Fundament-Entscheidungen (DB, ORM, Auth, UI-Basis, Env-Validierung) technisch verankern
3. Kern-Datenmodell nur für den ersten Anwendungsfall
4. Einen kompletten vertikalen Feature-Slice end-to-end bauen und als Muster reviewen
5. Auth/Multi-Tenancy vollständig ausbauen, basierend auf dem, was Schritt 4 gezeigt hat
6. Weitere Features nach demselben Muster, vertikal pro Person/Team aufgeteilt
7. Tests von Anfang an pro Feature mitliefern, nicht nachträglich
8. Observability & Deployment-Pipeline parallel vervollständigen

Der rote Faden: **so früh wie möglich ein komplettes, wenn auch minimales Feature end-to-end lauffähig haben**, statt lange an Infrastruktur oder Datenmodell zu feilen, bevor irgendetwas funktioniert. Das deckt Architekturfehler (z. B. bei der Multi-Tenancy-Autorisierung) auf, wenn sie noch billig zu beheben sind, statt wenn schon zwanzig Features darauf aufbauen.
