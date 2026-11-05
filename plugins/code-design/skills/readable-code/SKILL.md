---
name: readable-code
description: The user's way of writing TypeScript and React code that reads top to bottom without effort - explicit control flow, blank lines between steps, named derived values (useMemo) instead of logic buried in JSX, one component per file with `interface Props`, reusable helpers extracted to utils/ or hooks/, strict Single Responsibility and no prop drilling. Use this whenever you write, edit or review TypeScript, React, Next.js, React Native or NestJS code, even for small changes, and whenever the user says code is unreadable, messy, too clever, too compact, or asks to clean it up. For how to split pages into sections that own their data, also use component-architecture.
---

# Readable code

Code is read far more often than it is written. Optimise for the next person
scanning the file: every step visible, every name saying what it holds, every
file doing one thing. Shorter is not better when it hides what happens.

This is about the *way* of programming, not formatting alone. Prettier and the
linter handle spacing; the rules below are the choices a formatter cannot make.

## 1. One responsibility, everywhere

Single Responsibility is the rule everything else follows from: a function,
hook, component or file has **one reason to change**.

- A component renders one thing. When it also fetches, formats, filters and
  handles three kinds of clicks, split it.
- A function does one step. When its name needs "and" (`loadAndFormatOrders`),
  it is two functions.
- A file holds one thing (see section 4).

**No prop drilling.** A prop that a component only passes on to a child is a
smell, not a convenience. The component that needs the data gets it itself,
through a hook, context, store, server loader or the URL. See the
`component-architecture` skill for how to restructure a page this way.

**Self-contained components.** Every component should work on its own when you
drop it somewhere else: it takes the few values that make it *this* instance
(an id, a query, a label) and gets or derives everything else itself. Leaves
(rows, cards, badges) are pure: values in, callbacks out.

## 2. Explicit control flow

- Always use braces for `if`, `else`, `for` and `while`, also for one statement.
- No nested ternaries. Use an early return, a small named function, or a
  separate component. A single ternary for a simple choice in JSX is fine.
- Prefer early returns (guard clauses) over deep `if` nesting.
- No one-liners that hide control flow: no `&&`/`||` chains for side effects,
  no long `.map().filter().reduce()` chains inside an expression you then have
  to untangle. Break them into named steps.
- `null` and `undefined` each have their own meaning; use both for what they
  mean. `undefined`: not provided or not set (an optional prop, a missing key).
  `null`: deliberately empty (no selection, no result, a nullable column).
  Do not convert one into the other without a reason.

```ts
// Hard to scan
const label = order ? order.paid ? "Betaald" : order.overdue ? "Te laat" : "Open" : "-";

// Readable
function orderStatusLabel(order: Order | null): string {
  if (!order) {
    return "-";
  }

  if (order.paid) {
    return "Betaald";
  }

  if (order.overdue) {
    return "Te laat";
  }

  return "Open";
}
```

## 3. Blank lines separate steps

Group related statements and put a blank line between groups, so a function
reads as a list of steps:

- after reading params or props, before the next hooks;
- between the data hooks and the derived values;
- before and after an `if` block when other statements follow;
- before the final `return`.

```tsx
export function OrderSummary({ orderId }: Props) {
  const { data: order } = useOrder(orderId);
  const { formatPrice } = useCurrency();

  const total = useMemo(() => sumLines(order?.lines ?? []), [order]);

  if (!order) {
    return null;
  }

  return <SummaryCard title={order.number} total={formatPrice(total)} />;
}
```

## 4. Files: one thing per file

**One component per file**, named after the file. No exceptions for "small"
ones: a skeleton, an empty state, a row, a child item or a wrapper each get
their own file next to the component that uses them.

```
components/orders/
  order-list.tsx
  order-list-skeleton.tsx
  order-list-empty.tsx
  order-row.tsx
```

**A component file contains only the component and its `Props`.** No helper
functions, formatters or constants above or below the component:

- **Reusable function** (date/price/phone formatting, distance maths, sorting
  helpers) → `utils/`, grouped by concern: `utils/date.ts` with `formatDate`,
  `formatTime`, `formatDateRange`. Check `utils/` first; reuse before writing a
  new one.
- **Reusable stateful logic** (anything using hooks) → `hooks/use-<name>.ts`.
- **Logic only one component uses** → a file next to it
  (`order-list-utils.ts`, `order-list-data.ts`), not inside the component file.
- **Constants** → next to their owner in their own file (`order-list-config.ts`)
  or in `constants/` when shared.
- **Schemas** → `schemas/`, one schema and its inferred type per file.

## 5. `interface Props`

Declare a component's props as an `interface` named `Props`, in the component's
own file, directly above the component:

```tsx
interface Props {
  orderId: string;
  onSelect: (order: Order) => void;
}

export function OrderRow({ orderId, onSelect }: Props) {
  // ...
}
```

Not a `type` alias, not `OrderRowProps`. One exception: Next.js pages and
layouts use Next's route-checked `PageProps<"/route">` / `LayoutProps<"/route">`. When another file needs the props
type, export it (`export interface Props`) and import it with an alias where a
name clash would occur (`import type { Props as OrderRowProps } from "./order-row"`).

## 6. Name derived values instead of computing in JSX

A component body should read as: get data → derive values → render. Logic does
not belong inside JSX or in a long inline expression.

- Derive values into named constants before the `return`. Use `useMemo` when
  the derivation is non-trivial (filtering, sorting, grouping, mapping a list)
  or its result is passed to a memoised child; a plain `const` for simple
  lookups.
- Name event handlers (`handleSubmit`, `handleSelectOrder`) instead of
  multi-line arrow functions in JSX. Use `useCallback` when the handler goes to
  a memoised child or a dependency array.
- A derivation that is pure and could be reused (formatting, totals) lives in
  `utils/` and is called from the `useMemo`.

```tsx
// Hard to read: logic hidden in the JSX
return (
  <List>
    {orders
      .filter((o) => o.status !== "cancelled" && (!query || o.number.includes(query)))
      .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime())
      .map((o) => (
        <OrderRow key={o.id} orderId={o.id}
          onSelect={() => { select(o.id); router.push(`/orders/${o.id}`); }} />
      ))}
  </List>
);

// Readable: named steps, then render
const visibleOrders = useMemo(() => {
  const active = orders.filter((order) => order.status !== "cancelled");
  const matching = filterByNumber(active, query);

  return sortByNewest(matching);
}, [orders, query]);

return (
  <List>
    {visibleOrders.map((order) => (
      <OrderRow key={order.id} orderId={order.id} onSelect={openOrder} />
    ))}
  </List>
);
```

## 7. Names

- Say what a value holds, not its type: `visibleOrders`, not `arr2` or `data`.
- Booleans read as a question: `isLoading`, `hasAccess`, `canEdit`.
- Functions start with a verb: `formatDate`, `sortByNewest`, `openOrder`.
- Components: `<Feature><Role>` (`OrderList`, `OrderRow`, `OrderListSkeleton`).
- Files are kebab-case and match their export (`order-list-skeleton.tsx`).

## Checklist before you finish

- [ ] Every file holds one component, hook, schema or concern.
- [ ] No helper functions or constants inside a component file.
- [ ] Reusable formatters and helpers are in `utils/` or `hooks/`, and existing ones were reused.
- [ ] Props are `interface Props`.
- [ ] No prop is passed through a component that does not use it.
- [ ] No nested ternaries; every `if` has braces; steps separated by blank lines.
- [ ] Non-trivial derivations are named (`useMemo`), not inline in JSX.
- [ ] The project's format, lint and typecheck pass.

When the project's own `AGENTS.md` / `CLAUDE.md` says something different,
follow the project and mention the difference.
