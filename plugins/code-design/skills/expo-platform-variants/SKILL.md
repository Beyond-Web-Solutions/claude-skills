---
name: expo-platform-variants
description: How to build platform-specific component variants in Expo / React Native (.ios.tsx, .android.tsx, .web.tsx files, @expo/ui SwiftUI and Jetpack Compose) without duplicating data fetching or behaviour. Use this whenever you add, split or refactor a component into .ios.tsx / .android.tsx / .web.tsx files, when platform files fetch the same data twice or drift apart in props, when an import from the base file is undefined at runtime, or when the user asks how to do native iOS vs Android UI in Expo. Builds on component-architecture and readable-code (load those too for the general SRP, file and naming rules).
---

# Expo platform variants

Platform files let iOS and Android render natively (SwiftUI via
`@expo/ui/swift-ui`, Jetpack Compose via `@expo/ui/jetpack-compose`) while the
rest of the app stays shared. The rule: **platform files differ only in
rendering**. Data, behaviour and the props contract are shared.

The layering (route → screen → section → leaf) comes from the
`component-architecture` skill. This skill adds what changes when each of those
has platform variants.

## File set

For a component `foo` that needs platform variants:

| File | Role |
|---|---|
| `foo.tsx` | Default (web or fallback; often `return null`) **and the contract**: `export interface Props` |
| `foo.ios.tsx` | iOS rendering, same props |
| `foo.android.tsx` | Android rendering, same props |
| `foo-data.ts` | Colocated data hook used by every variant |
| `foo-size.ts` / `foo-constants.ts` | Shared constants, if the variants need them |

Metro picks the platform file at build time; callers always
`import { Foo } from "./foo"`.

## The import pitfall

Inside `foo.ios.tsx`, `import { X } from "./foo"` resolves to `foo.ios.tsx`
itself, so `X` is `undefined` at runtime (TypeScript does not catch this,
because it type-checks against `foo.tsx`).

- Only use `import type { Props } from "./foo"` from the base name. The
  platform file does not declare its own `Props`; it uses the base file's.
- Put shared values (constants, helpers) in a separate file without platform
  variants (`foo-size.ts`) and import from there.

## What is shared, what differs

Shared across all variants (never written twice):
- the **data hook** (`useNearbyLocations` in `nearby-locations-data.ts`);
- the **action hook** (`usePickOnMap`);
- the **props** (`export interface Props` in the base file);
- formatting and maths (`distanceInMeters`, `formatDistance` in `utils/`).

Differs per platform:
- layout and native components (`List`/`Section` in SwiftUI,
  `Column`/`Surface` with modifiers in Compose);
- platform-specific affordances (`ContentUnavailableView`, Material typography).

If two platform files each call the same fetch or build the same callback, the
shared hook is missing. Extract it before adding anything else.

## Which level to split

Split at the lowest level that actually differs. Usually that is the screen
(native list vs column) and the leaves (rows, cards). Sections often need
variants only because they render a native container; their data and action
hooks still come from one place. A route file (`app/*.tsx`) should almost never
need platform variants.

## Checklist for a new variant set

1. Write the base file with the props type and a default implementation.
2. Write the data hook in `<name>-data.ts`; both variants call it.
3. Reuse or add an action hook in `hooks/` for what happens on tap.
4. Write `.ios.tsx` and `.android.tsx` with identical props, importing only
   types from the base name.
5. Typecheck, then run on both platforms (or at least check the imports by
   hand: no value imports from the base name).

## Smells and the fix

| Smell | Fix |
|---|---|
| Two platform files both calling the same fetch | Colocated data hook used by both |
| Same `onPress` logic written in both variants | Shared action hook |
| A value import from the base name in a platform file (`undefined` at runtime) | `import type`, move values to a separate file |
| iOS and Android variants with different props | One props type in the base file |
| Platform variants of a route file | Move the difference down into the screen or a section |

## Worked example

For a complete before/after of a search screen with iOS (SwiftUI) and Android
(Compose) variants, read `references/example-refactor.md`.
