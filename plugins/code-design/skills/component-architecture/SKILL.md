---
name: component-architecture
description: How to structure React UI code (React, Next.js, Remix, React Native, Expo) so each component has one job and owns its own data (Single Responsibility Principle, container/presentational components, no prop drilling, colocation, clear naming). Use this whenever you create or refactor pages, routes, screens, sections, lists or cards; whenever a page or screen component takes more than a handful of props or threads callbacks and data down several levels; whenever the user says components are messy, ugly, low quality, "not nice", too big, or asks about SRP, SOLID, atomic design, smart/dumb components, prop drilling, colocation or how to split a component. For .ios.tsx / .android.tsx platform variants in Expo, also use expo-platform-variants.
---

# Component architecture

The goal: a page or screen reads like a table of contents. Each section on it
is a component that knows how to get its own data and what to do when clicked
or tapped, and the small visual pieces inside a section are pure: data in,
callbacks out.

This skill exists because the opposite keeps happening: a route or page
collects every piece of data and every handler at the top, then passes ten
props down through components that only forward them. That version works, but
every change touches three files and the page is unreadable.

These rules hold for any React renderer. Where a framework changes the details
(Next.js server components, Expo platform files) that is called out below.

## Vocabulary (so you and the user mean the same thing)

- **Single Responsibility Principle (SRP)**: a module has one reason to change.
  The "S" in SOLID. This is the principle the structure below follows.
- **Container / presentational** (also smart/dumb): a container component
  fetches data and decides what happens; a presentational component only renders
  what it is given. Hooks and server components blurred the line, but the split
  is still the right mental model.
- **Prop drilling**: passing data or callbacks through components that do not
  use them, only to reach a component further down. The main smell this skill
  removes.
- **Colocation**: keep a section's data hook, types and helpers next to the
  section that uses them, not in a global `utils` pile.
- **Self-contained (autonomous) components**: a component that works wherever
  you drop it, because it takes only the values that identify *this* instance
  and gets everything else itself. This is what the user means by "atomic"
  components, and it is the goal of the three layers below.
- **Atomic design**: a naming scheme (atoms, molecules, organisms, templates,
  pages) for *how big* a component is. It says nothing about who owns data. Do
  not confuse it with SRP or with self-contained components; you can use its
  names, but the ownership rules below are what matter.

## The three layers

Keep every component in exactly one of these roles.

### 1. Routes, pages and screens: composition only

A route file (`app/orders/page.tsx` in Next.js, `app/search.tsx` in Expo
Router, a route module in Remix) renders one page/screen. A page places
sections. Neither fetches data for a section or builds handlers for it.

What a page may do: hold UI state that belongs to the page as a whole (the
current search query, a selected tab, a URL param) and pass that one value
down. If a page component ends up with more than three or four props, a
section is missing.

```tsx
// app/search/page.tsx (Next.js) — the query lives in the URL
interface Props {
  searchParams: Promise<{ q?: string }>;
}

export default async function SearchPage({ searchParams }: Props) {
  const { q } = await searchParams;
  return (
    <main>
      <SearchBox />
      {q ? <PlaceResults query={q} /> : (
        <>
          <FeaturedLocations />
          <NearbyLocations />
        </>
      )}
    </main>
  );
}
```

### 2. Sections: own data and behaviour

A section is a self-contained block of a page: "featured venues", "nearby
pick-up points", "search results". It takes zero or one props (the query, an
id) and gets everything else itself:

- a **data hook** colocated with the section (`useNearbyLocations` in
  `nearby-locations-data.ts`), which wraps the store, API or static data. In a
  Next.js server component the section can simply `await` a colocated data
  function (`getFeaturedLocations()`) instead;
- an **action hook** shared across sections when several do the same thing on
  click (`usePickOnMap` = select on the map and navigate).

A section renders nothing when it has nothing to show (no position known, no
featured items) so the page does not need to know about that case.

```tsx
export function NearbyLocations() {
  const nearby = useNearbyLocations();     // data: position + nearest locations
  const { pickLocation } = usePickOnMap(); // behaviour: select + navigate

  if (nearby.length === 0) {
    return null;
  }

  return (
    <Section title="Nearby">
      {nearby.map(({ location, distanceMeters }) => (
        <LocationResultRow key={location.id} location={location}
          distanceMeters={distanceMeters} onPress={() => pickLocation(location)} />
      ))}
    </Section>
  );
}
```

### 3. Leaves: pure rendering

Rows, cards, badges, buttons. They receive plain values and callbacks and have
no data hooks. They may use theme/style hooks. They know nothing about
routing or stores, so they are reusable in a list, a dialog/sheet and a
carousel.

```tsx
interface Props {
  location: Location;
  distanceMeters: number | null;
  onPress: () => void;
}

export function LocationResultRow({ location, distanceMeters, onPress }: Props) { ... }
```

## Framework notes

- **Next.js App Router**: sections that only show data are server components
  that fetch it themselves (`await getX()` in the section, cached with
  `cache`/`fetch` so two sections asking for the same data is fine). Interactive
  bits become small client components (`"use client"`) at the leaf or section
  level, never the whole page. Wrap slow sections in `<Suspense>` at the page so
  each streams on its own. Mutations go through a server action colocated with
  the section, not a callback passed from the page.
- **Client-only React / React Native**: sections use data hooks (React Query,
  SWR, a store, or a plain `useEffect` hook) as shown above.
- **Expo / React Native platform files** (`.ios.tsx`, `.android.tsx`): see the
  `expo-platform-variants` skill. The layering here stays the same; platform
  files only change the rendering.

## Where shared things go

- **Behaviour used by several sections** → a hook in `hooks/`
  (`usePickOnMap`, `useCurrentPosition`). Never pass the same callback down
  from the page into three sections.
- **State shared between pages** (what the map is focused on, auth, cart) →
  a context, store or the URL, read where needed. Do not thread it through
  props.
- **Static data** → `data/`; **remote calls** → `services/` (or `lib/`);
  **pure helpers** (distance, formatting) → `utils/`. Sections import these;
  pages do not.
- **A section's own data hook or loader** → next to the section, as
  `<section>-data.ts`, unless two sections share it (then `hooks/` or `lib/`).

## Files and naming

- One component per file, always. Skeletons, empty states, rows and child
  items each get their own file next to their parent
  (`order-list.tsx`, `order-list-skeleton.tsx`, `order-row.tsx`).
- A component file holds only `interface Props` and the component. Helper
  functions, formatters and constants go elsewhere: reusable ones in `utils/`
  or `hooks/`, single-use ones in a file next to the component. The
  `readable-code` skill has the full rules.
- Props are always `interface Props` in the component's file, never a `type`
  alias or `<Component>Props`.
- Name by feature and role, not by size: `FeaturedLocations` (section),
  `FeaturedLocationCard` (leaf), `useNearbyLocations` (data hook),
  `PlaceResults` (section taking `query`). Avoid generic names like `List`,
  `Item`, `Container`.
- Group components by feature folder (`components/search/`,
  `components/map/`), not by type (`components/cards/`).

## Refactoring procedure

When a page or route has grown a wide props bag, do this in order rather
than patching props one by one:

1. **List the data sources** the page collects (positions, fetched results,
   static lists, store values).
2. **List the actions** (on click, on select, on dismiss) and what they do.
3. **Draw the sections** the user sees on the page. Each data source should
   map to one section; a data source used by several sections becomes a shared
   hook or loader.
4. **Move each data source into a hook or loader next to its section**, and
   each action into a shared action hook (or server action) if more than one
   section triggers it.
5. **Make the leaves pure**: strip data hooks and routing from rows and cards;
   they take values and `onPress`/`onClick`.
6. **Collapse the page**: it should now render sections with zero or one
   prop each. Delete the props type that listed everything.

Keep behaviour identical while doing this; a structure refactor should not
change what the user sees. Run the typecheck and lint when done.

## Smells and the fix

| Smell | Fix |
|---|---|
| Page/route component with 5+ props | Split into sections that fetch their own data |
| A callback created in the route and passed through 2+ components | Shared action hook or server action |
| A whole page marked `"use client"` to make one button work | Push `"use client"` down to the leaf |
| A leaf (row, card) calling the router or a store | Lift to the section via an `onPress`/`onClick` prop |
| `utils/` holding something only one section uses | Move it next to that section |
| Second component declared in a file "because it is small" | Own file |
| Generic names (`Item`, `Card2`, `ListContainer`) | `<Feature><Role>` names |

## Worked example

For a complete before/after of a search page (route with eight props →
three self-contained sections), read `references/example-refactor.md`. Read it
when the user's code looks like the "before" and you want to show the target
shape. For the same example with iOS/Android variants, see the
`expo-platform-variants` skill.
