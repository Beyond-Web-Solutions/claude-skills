---
name: nextjs-data-streaming
description: How to load data in Next.js App Router (15/16, Cache Components) - fetch in the server component that renders the data (the lowest level), wrap exactly that part in <Suspense> with a matching skeleton component, keep pages synchronous, and pass params / searchParams / cookies down as promises instead of awaiting them at the top. Use this whenever you write or change a Next.js page, layout or server component that loads data, reads params or searchParams, checks the session, adds a loading state, skeleton, loading.tsx or Suspense boundary, or when a page is slow, blocks on one query, shows a full-page spinner, or the build reports a blocking route / uncached data outside Suspense.
---

# Next.js data streaming

The goal: the page shell (layout, headings, card frames, static text) renders
instantly, and every piece of data streams in on its own, in the exact spot
where it is shown, with a skeleton that has the same shape.

That only works when nothing above a data component waits. Every `await` at
the top of a page or layout (data, `params`, `searchParams`, `cookies()`, the
session) blocks everything below it. So the rule is: **await as low as
possible, suspend as tight as possible.**

This skill builds on `component-architecture` (sections own their data) and
`readable-code` (one component per file, `interface Props`). Before writing
code, check the installed Next.js version and read the matching guide in
`node_modules/next/dist/docs/` (streaming, fetching data, cache components).

## 1. Pages and layouts do not await

A page is a synchronous function that places sections. It does not `await`
data, the session, `params` or `searchParams`, and is not `async`.

```tsx
// app/orders/page.tsx
export default function OrdersPage({ searchParams }: PageProps<"/orders">) {
  return (
    <main>
      <h1>Orders</h1>

      <OrderFilters />

      <Suspense fallback={<OrderTableSkeleton />}>
        <OrderTable searchParams={searchParams} />
      </Suspense>
    </main>
  );
}
```

`<h1>` and `<OrderFilters />` are part of the static shell. Only the table
waits. With `cacheComponents: true`, an await outside a Suspense boundary is a
build error (blocking route), which is the framework enforcing this rule.

## 2. Fetch in the component that renders the data

The component that shows a value is the one that awaits it. Not its parent,
not the page.

- Each data component calls its own query function (`getOrders(filters)`),
  from `data/queries.ts` of its feature. Never `fetch` or query the database
  inline in a component.
- Two components needing the same data both call the query. Wrap query
  functions in React `cache()` so it runs once per request; no prop drilling
  of fetched data.
- Independent data goes in **sibling** components with their own boundaries,
  so they load in parallel. Only nest awaits when one really depends on the
  other.

## 3. Suspend only the dynamic part

Put the `<Suspense>` boundary around the smallest piece that needs the data.
Static chrome around it (card, title, description, labels) stays outside so it
renders in the shell.

```tsx
// features/orders/components/open-orders-card.tsx
export function OpenOrdersCard() {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Open orders</CardTitle>
        <CardDescription>Orders that have not been shipped yet.</CardDescription>
      </CardHeader>

      <CardContent>
        <Suspense fallback={<OpenOrdersCountSkeleton />}>
          <OpenOrdersCount />
        </Suspense>
      </CardContent>
    </Card>
  );
}

// features/orders/components/open-orders-count.tsx
export async function OpenOrdersCount() {
  const count = await getOpenOrdersCount();

  return <p className="text-4xl font-bold">{count}</p>;
}
```

The async component (`OpenOrdersCount`) gets its own file, like every
component. It is not declared under the card.

## 4. Every boundary gets a matching skeleton component

- One skeleton per async component, in its own file next to it:
  `open-orders-count.tsx` → `open-orders-count-skeleton.tsx`,
  `OpenOrdersCountSkeleton`.
- The skeleton has the **same outer size and layout** as the real content
  (same row count, column widths, heights), built from the shared `Skeleton`
  primitive. This prevents layout shift when the data arrives.
- The skeleton only replaces what is inside the boundary. If the card frame is
  outside Suspense, the skeleton does not draw a card.
- No generic spinners or "Loading…" text as fallback, and no `fallback={null}`
  unless the component really renders nothing visible while loading.

## 5. Passing params and searchParams down

`params` and `searchParams` are promises. Do not destructure them in the
page; pass the promise to the component that needs the value, and await it
there, inside its Suspense boundary.

```tsx
// features/orders/components/order-table.tsx
interface Props {
  searchParams: Promise<Record<string, string | string[] | undefined>>;
}

export async function OrderTable({ searchParams }: Props) {
  const filters = orderFiltersSchema.parse(await searchParams);

  const orders = await getOrders(filters);

  if (orders.length === 0) {
    return <OrderTableEmpty />;
  }

  return <DataTable rows={orders} columns={orderColumns} />;
}
```

- Type the prop as the promise. For route params use the exact shape
  (`Promise<{ orderId: string }>`).
- Validate search params with a zod schema in `schemas/`
  (`schemas/order-filters.ts`). It turns `string | string[] | undefined` into
  the typed filters the query needs, treats empty strings as absent and fills
  defaults. Never trust raw search params in a query.
- A **client** component that needs the value receives the same promise and
  unwraps it with `use(searchParams)` inside a Suspense boundary. For
  interactive filters that read and update the URL, use `useSearchParams()` /
  `router.replace` in a client leaf instead.
- Several components reading the same params each receive the promise; that
  is fine, the promise resolves once.
- `notFound()` for a missing record belongs in the component that fetched it.

## 6. Data Access Layer: auth lives in the query

All data access goes through a Data Access Layer (DAL): the query and
mutation functions in `features/<feature>/data/`. **Every DAL function checks
the session and the permission itself**, as its first step, before it touches
the database. Pages, layouts and components never call `requireSession()` or a
permission check.

```ts
// features/orders/data/queries.ts
import "server-only";

export const getOrders = cache(async (filters: OrderFilters) => {
  const session = await requireSession();

  await requirePermission(session, "orders:read");

  return db.query.orders.findMany({ where: ordersWhere(filters) });
});
```

Why here and not at the top of the page:
- Reading the session reads cookies, which is dynamic. An `await` at the top
  of a page or layout blocks the whole segment; inside the query it only
  blocks the component that needs the data, inside its own boundary.
- The check travels with the data: a component is protected wherever it is
  used, and a new page cannot forget the check.

Rules:
- `requireSession()` and permission helpers are wrapped in `cache()`, so ten
  queries on one page read the session once.
- DAL files start with `import "server-only"` so they can never end up in a
  client bundle.
- Return only what the UI needs (a narrow object), not raw rows with
  sensitive columns.
- Mutations (server actions, route handlers) follow the same rule: check the
  session and permission first, in the function itself.
- Components that show the user (avatar, name) call `getCurrentUser()` inside
  their own Suspense boundary.
- A coarse redirect for a whole area (not signed in → sign-in page) may live
  in `proxy.ts`. That is a UX shortcut, not the security check; the DAL check
  stays.

## 7. Static data: cache it into the shell

Data that does not depend on the request (settings, product lists, content)
does not need a skeleton at all: mark the query or component with `"use cache"`
(and `cacheLife` / `cacheTag` when it must refresh) so it is prerendered into
the static shell. Only request-dependent data streams.

## 8. No `loading.tsx`

Do not add `loading.tsx`. It wraps the whole page in one boundary, so the
whole page is replaced by one skeleton and the shell you built is hidden.
Use a `<Suspense>` boundary with its own skeleton per section instead. The
only exception is a page that has nothing at all to show without data; say so
when you add one.

When you touch a route that has a `loading.tsx` (often pointing at a shared
`page-skeletons.tsx`), replace it with per-section boundaries and skeleton
files, and remove the page skeleton once nothing uses it.

## 9. Page props

Pages type their props with Next's route-checked helper
`PageProps<"/orders/[orderId]">` (and layouts with `LayoutProps<...>`). This is
the one exception to `interface Props`; every other component, including the
sections a page renders, uses `interface Props`.

## Smells and the fix

| Smell | Fix |
|---|---|
| `export default async function Page` with `await` at the top | Synchronous page; move the await into a section |
| `const { id } = await params` in the page | Pass `params` to the section that needs `id` |
| `await requireSession()` at the top of a page or layout | Session and permission check inside the DAL function |
| `loading.tsx` in a route | Suspense per section with its own skeleton |
| One Suspense around the whole page or a whole card | Boundary around only the data part; chrome outside |
| Async child component declared under its parent in the same file | Own file, with its own skeleton file |
| `fallback={<p>Loading…</p>}` or a spinner | Skeleton component with the same shape |
| Several skeletons in one `page-skeletons.tsx` | One skeleton per file, next to its component |
| Two sequential awaits for unrelated data | Two sibling components, two boundaries |
| Query result passed down as a prop through several components | Each component calls the cached query itself |
| Raw `searchParams.status as Status` | zod schema in `schemas/` |

## Checklist

- [ ] No page or layout awaits data, params, searchParams or the session at the top.
- [ ] Every data component awaits its own data and its own params promise.
- [ ] Every Suspense boundary wraps only the dynamic part and has a matching skeleton in its own file.
- [ ] Search params are parsed with a schema.
- [ ] Every DAL query and mutation checks the session and permission itself; no page, layout or component does.
- [ ] No `loading.tsx`; pages use `PageProps<"/route">`, everything else `interface Props`.
- [ ] `next build` shows no blocking-route errors and the static shell contains the page chrome.
