# Ein großes Next.js/ERP-Projekt von Null starten – überarbeitete Version

Beispielprojekt: ein B2B-ERP mit Organisationen, Kunden und Rechnungen (Customer → Invoice). Diese Version übernimmt die Korrekturen aus der Diskussion: Auth/Multi-Tenancy von Anfang an, Drizzle statt Prisma, eine schlanke Use-Case-Schicht zwischen Server Action und Datenbank, feature-basierte statt technische Ordnerstruktur, kein Monorepo ohne echten Bedarf, und ein Business-Beispiel statt eines generischen Ticket-Systems.

Roter Faden bleibt: **minimale Architektur → echtes Feature → Muster erkennen → Architektur gezielt verbessern → nächstes Feature.** Nicht Architektur → Infrastruktur → Architektur → Features.

## Schritt 1: Repository & Grundgerüst – bewusst kein Monorepo am Tag 1

```bash
npx create-next-app@latest erp \
  --typescript --tailwind --eslint --app --src-dir \
  --import-alias "@/*"
cd erp
git init && git add -A && git commit -m "chore: initial next.js scaffold"
```

Struktur bleibt zunächst ein einzelnes Repo:

```
erp/
  src/
    app/
    features/
    server/
    lib/
    components/ui/
  drizzle/
```

Turborepo/Monorepo erst dann einführen, wenn tatsächlich eine zweite App entsteht (Admin-Panel, separate API, Mobile-Backend) – nicht auf Vorrat. Der Umbau von Single-Repo zu Monorepo ist mechanisch und günstig; die vorzeitige Komplexität eines Monorepos ohne zweite App ist es nicht.

Sofort einrichten, wie zuvor: Node-Version fixieren, ESLint/Prettier + Husky/lint-staged, Conventional Commits, CI-Skelett (`install → lint → typecheck → build`).

## Schritt 2: Datenbank, Drizzle, Migrations

```bash
npm i drizzle-orm postgres
npm i -D drizzle-kit
```

```ts
// drizzle/schema.ts
import { pgTable, text, timestamp, uuid, pgEnum } from 'drizzle-orm/pg-core'

export const roleEnum = pgEnum('role', ['OWNER', 'ADMIN', 'MEMBER'])

export const organizations = pgTable('organizations', {
  id: uuid('id').defaultRandom().primaryKey(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
})

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  name: text('name'),
})

export const memberships = pgTable('memberships', {
  id: uuid('id').defaultRandom().primaryKey(),
  userId: uuid('user_id').notNull().references(() => users.id),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  role: roleEnum('role').notNull().default('MEMBER'),
})
```

```bash
npx drizzle-kit generate
npx drizzle-kit migrate
```

Warum Drizzle hier passt: bei einem ERP kommen früh Berichte, Summenbildungen und Joins über mehrere Aggregate (Rechnungen, Positionen, Zahlungen) dazu – dafür will man nah an SQL bleiben können, statt gegen die Abstraktion eines ORM zu arbeiten.

## Schritt 3: Auth & Multi-Tenancy sofort, nicht "danach"

Das ist die wichtigste Korrektur gegenüber der ersten Version. Bevor das erste fachliche Feature entsteht, steht die Autorisierungskette:

```ts
// src/server/auth.ts
import { auth } from './auth-config' // z. B. Auth.js
import { db } from './db'
import { memberships } from '../../drizzle/schema'
import { and, eq } from 'drizzle-orm'

export async function requireAuth() {
  const session = await auth()
  if (!session?.user) throw new UnauthorizedError()
  return session.user
}

export async function requireMembership(organizationId: string) {
  const user = await requireAuth()
  const [membership] = await db
    .select()
    .from(memberships)
    .where(
      and(
        eq(memberships.userId, user.id),
        eq(memberships.organizationId, organizationId),
      ),
    )
  if (!membership) throw new ForbiddenError()
  return { user, membership }
}

export async function requireRole(organizationId: string, roles: Role[]) {
  const { membership, user } = await requireMembership(organizationId)
  if (!roles.includes(membership.role)) throw new ForbiddenError()
  return { user, membership }
}
```

Diese drei Funktionen existieren, bevor das erste fachliche Feature geschrieben wird, und **jede** Server Action und jede datenzugreifende Server Component ruft eine davon auf. Die Tenant-Grenze ist damit ein strukturelles Merkmal des Systems von Tag 1 an, nicht eine Ergänzung, die man nachträglich in bestehende Queries einbaut – genau dort passieren die teuersten Fehler (Organisation A sieht Daten von Organisation B), wenn man sie nachträglich einzieht.

Middleware (`middleware.ts`) übernimmt zusätzlich den groben Schutz nicht eingeloggter Zugriffe auf `/app/*`, ersetzt aber nicht die serverseitigen Checks oben.

## Schritt 4: Feature-basierte Struktur statt technischer Schichten

```
src/features/
  customers/
    components/
    actions.ts        # Server Actions (Eingangspunkt für Mutationen)
    use-cases.ts       # Geschäftslogik, unabhängig von Next.js
    queries.ts         # Lesezugriffe für Server Components
    schema.ts          # Zod-Schemas für Validierung
    types.ts
  invoices/
    components/
    actions.ts
    use-cases.ts
    queries.ts
    schema.ts
    types.ts
```

Jede Domäne (Kunden, Rechnungen, später Produkte, Bestellungen, Buchhaltung) ist ein eigenständiger, in sich geschlossener Ordner. Das erlaubt vertikale Aufteilung im Team: eine Person kann "Rechnungen" komplett von Use-Case bis UI übernehmen, ohne dass mehrere Leute in denselben `components/`- oder `actions/`-Sammelordnern kollidieren.

## Schritt 5: Eine schlanke Use-Case-Schicht – ohne Clean-Architecture-Overkill

Bewusst **kein** Stack aus Controller/Service/Repository/Factory/Mapper/DTO/Entity/Interactor von Anfang an. Stattdessen eine einzige zusätzliche Schicht zwischen Server Action und Datenbank: einfache, testbare Funktionen, die die Geschäftsregeln kapseln.

```ts
// features/invoices/use-cases.ts
import { db } from '@/server/db'
import { invoices, invoicePositions } from '../../../drizzle/schema'

export async function createInvoice(input: {
  organizationId: string
  customerId: string
  positions: { description: string; quantity: number; unitPrice: number }[]
}) {
  const total = input.positions.reduce(
    (sum, p) => sum + p.quantity * p.unitPrice,
    0,
  )

  return db.transaction(async (tx) => {
    const [invoice] = await tx
      .insert(invoices)
      .values({
        organizationId: input.organizationId,
        customerId: input.customerId,
        status: 'DRAFT',
        total,
      })
      .returning()

    await tx.insert(invoicePositions).values(
      input.positions.map((p) => ({ ...p, invoiceId: invoice.id })),
    )

    return invoice
  })
}
```

```ts
// features/invoices/actions.ts
'use server'

import { z } from 'zod'
import { revalidatePath } from 'next/cache'
import { requireRole } from '@/server/auth'
import { createInvoice } from './use-cases'

const schema = z.object({
  organizationId: z.string().uuid(),
  customerId: z.string().uuid(),
  positions: z
    .array(
      z.object({
        description: z.string().min(1),
        quantity: z.coerce.number().positive(),
        unitPrice: z.coerce.number().positive(),
      }),
    )
    .min(1),
})

export async function createInvoiceAction(formData: FormData) {
  const parsed = schema.safeParse(JSON.parse(formData.get('payload') as string))
  if (!parsed.success) return { error: parsed.error.flatten() }

  await requireRole(parsed.data.organizationId, ['OWNER', 'ADMIN'])
  const invoice = await createInvoice(parsed.data)
  revalidatePath(`/app/invoices`)
  return { invoice }
}
```

Der Punkt dieser Trennung: `createInvoice()` kennt weder Next.js noch FormData noch Autorisierung – sie lässt sich isoliert unit-testen (Summenberechnung, Transaktionslogik) und bleibt stabil, auch wenn sich später herausstellt, dass Rechnungen auch per Cronjob oder Import-Skript erzeugt werden müssen, nicht nur über die UI. Ein generisches Repository-Interface darüber wird bewusst *nicht* eingeführt, solange es nur eine Datenquelle (Postgres) gibt und keine Notwendigkeit für austauschbare Persistenz oder In-Memory-Tests besteht – das käme erst, wenn ein zweiter, konkreter Grund dafür auftaucht.

## Schritt 6: Der erste vertikale Slice – ein echter Business-Flow

Statt eines generischen Ticket-Beispiels der ERP-typische Kern-Flow:

```
Login → Organisation wählen → Kunde anlegen → Rechnung erstellen → Rechnung anzeigen
```

```
src/app/
  (app)/
    layout.tsx                          # Org-Auswahl, Navigation
    orgs/[orgId]/
      customers/
        page.tsx                        # Liste + neuer Kunde
      invoices/
        page.tsx                        # Liste
        [invoiceId]/page.tsx            # Detailansicht
```

```tsx
// app/(app)/orgs/[orgId]/invoices/page.tsx
import { requireMembership } from '@/server/auth'
import { listInvoices } from '@/features/invoices/queries'
import { NewInvoiceForm } from '@/features/invoices/components/new-invoice-form'

export default async function InvoicesPage({
  params,
}: {
  params: Promise<{ orgId: string }>
}) {
  const { orgId } = await params
  await requireMembership(orgId)
  const invoices = await listInvoices(orgId)

  return (
    <div>
      <h1 className="text-xl font-semibold">Rechnungen</h1>
      <ul>
        {invoices.map((i) => (
          <li key={i.id}>{i.id} — {i.total} CHF — {i.status}</li>
        ))}
      </ul>
      <NewInvoiceForm organizationId={orgId} />
    </div>
  )
}
```

Nach diesem Slice ist bereits real getestet: Auth, Multi-Tenancy-Grenze, Routing, Formulare inkl. Validierung, Server Actions, Use-Case-Logik mit Geschäftsrechnung (Summenbildung), Autorisierung nach Rolle, UI. Das ist deutlich aussagekräftiger als 15 einzelne Infrastruktur-Häppchen, weil es alle Schichten im echten Zusammenspiel zeigt – inklusive der Stellen, an denen sie in der Praxis reiben (z. B. wie Formulardaten mit verschachtelten Positionen sauber validiert werden).

## Schritt 7: Tests ab dem ersten Business-Feature, aber schlank

Kein "Test-Nachholprojekt" später, sondern von Anfang an drei Kategorien, bewusst minimal gehalten:

```ts
// Unit: reine Geschäftslogik
test('Rechnungssumme wird korrekt berechnet', () => {
  // createInvoice-Berechnung isoliert testen, ohne DB
})

// Integration: DB + Autorisierung zusammen
test('User aus Organisation A kann keine Rechnung aus Organisation B sehen', async () => {
  // requireMembership() muss ForbiddenError werfen
})

// E2E: kritischer Flow
test('Login → Kunde anlegen → Rechnung erstellen → Rechnung sichtbar', async () => {
  // Playwright
})
```

Genau diese drei Tests schützen von Anfang an exakt die Stelle, an der ein Fehler am teuersten wäre: die Tenant-Grenze und die Kernberechnung. Weitere Tests kommen erst dazu, wenn weitere Features echte Komplexität mitbringen.

## Schritt 8: Muster erkennen, dann gezielt Architektur verbessern

Nach dem ersten Slice (Schritt 6) und ggf. einem zweiten Feature (z. B. Zahlungen erfassen) explizit innehalten und prüfen:

- Wiederholt sich ein Muster in `use-cases.ts` über mehrere Features (z. B. immer dieselbe Art von Transaktions-Handling)? → ggf. einen gemeinsamen Helfer extrahieren.
- Werden Autorisierungs-Checks konsistent verwendet oder gibt es Stellen, an denen sie vergessen wurden? → ggf. per Lint-Regel oder Wrapper erzwingen.
- Wird ein Repository-Layer tatsächlich gebraucht (z. B. weil Tests ohne echte DB laufen sollen)? Erst jetzt einführen, mit konkretem Anlass – nicht vorab spekulativ.

Diese Refactoring-Runde findet **nach** echten Features statt, nicht davor – das ist der Kernunterschied zur Clean-Architecture-Falle, in die man tappt, wenn man Repository/Service/DTO-Schichten baut, bevor man weiß, welche Abstraktion man tatsächlich braucht.

## Schritt 9: Observability & Deployment-Pipeline vervollständigen

Wie in der ersten Version, aber jetzt mit echten Daten aus einem laufenden Business-Feature statt aus einem Spielzeugbeispiel:

- Sentry (oder vergleichbar) für Fehler in Server Actions und Use-Cases
- Preview-Deployments pro PR
- Migrations als expliziter, kontrollierter CI/CD-Schritt (Drizzle-Migrationen nie automatisch "mitlaufen lassen")
- Metriken für die geschäftskritischen Use-Cases (z. B. wie lange `createInvoice` dauert, Fehlerquote)

## Zusammengefasste, überarbeitete Reihenfolge

1. Repository + Next.js, kein Monorepo ohne echten Bedarf
2. PostgreSQL + Drizzle + Migrationen
3. Auth + Organisation + Membership + Rollen-Checks – vor dem ersten fachlichen Feature
4. Feature-basierte Ordnerstruktur anlegen
5. Schlanke Use-Case-Schicht einführen (keine vollständige Clean Architecture)
6. Erstes echtes Business-Feature: Kunde anlegen → Rechnung erstellen → Rechnung anzeigen
7. Direkt begleitende, schlanke Tests (Unit, Integration, ein E2E-Kernflow)
8. Zweites Business-Feature, dann Muster erkennen und Architektur gezielt verbessern
9. CI/CD & Observability vervollständigen
10. Monorepo erst einführen, wenn tatsächlich eine zweite App entsteht

Der entscheidende Unterschied zur ersten Version: Die Tenant-Grenze und eine minimale Geschäftslogik-Schicht sind von Anfang an strukturelle Bestandteile, nicht nachträgliche Ergänzungen – während zusätzliche Architektur-Schichten (Repository, DTOs, generische Abstraktionen) bewusst erst dann entstehen, wenn ein zweites oder drittes Feature einen konkreten Grund dafür liefert.
