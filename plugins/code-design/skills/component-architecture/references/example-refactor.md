# Worked example: a search page

A page shows featured venues, the nearest pick-up points, and place search
results while typing. Picking anything selects it on the map and navigates to
the map. The code is plain React with hooks, so it applies to a client page in
Next.js, a Vite app, or a single-platform React Native screen. A server
component variant follows at the end.

## Before: the route collects everything

```tsx
// app/search/page.tsx
"use client";
export default function SearchPage() {
  const [query, setQuery] = useState("");
  const router = useRouter();
  const { selectPlace, selectLocation } = useMapSelection();
  const userPosition = useCurrentPosition();
  const { places, searching } = usePlaceSearch(query, userPosition);
  const nearby = useMemo(
    () => (userPosition ? nearestLocations(userPosition, LOCATIONS, 5).map((e) => e.location) : []),
    [userPosition],
  );
  const onSelectPlace = useCallback((place: Place) => { selectPlace(place); router.push("/map"); }, [selectPlace, router]);
  const onSelectLocation = useCallback((location: Location) => { selectLocation(location); router.push("/map"); }, [selectLocation, router]);

  return (
    <Search
      query={query}
      onQueryChange={setQuery}
      places={places}
      searching={searching}
      featured={FEATURED_LOCATIONS}
      nearby={nearby}
      userPosition={userPosition}
      onSelectPlace={onSelectPlace}
      onSelectLocation={onSelectLocation}
    />
  );
}
```

`Search` destructured all of these, computed distances for rows, and wired the
same callbacks into rows. Adding a section meant touching the route, the props
type and the screen.

Problems, in the skill's terms:
- The route is a container for three unrelated things (featured, nearby,
  place search). It has three reasons to change.
- `onSelectLocation` is drilled: route → page → section → row.
- `userPosition` is drilled only so rows can show a distance.
- The whole page is a client component because of a few click handlers.

## After: route and page compose, sections own their data

```tsx
// app/search/page.tsx — one piece of page-level state, in the URL
interface Props {
  searchParams: Promise<{ q?: string }>;
}

export default async function SearchPage({ searchParams }: Props) {
  const { q } = await searchParams;
  return (
    <main>
      <SearchBox />                     {/* client leaf: writes ?q= */}
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

### Shared action hook

```tsx
// hooks/use-pick-on-map.ts
"use client";
export function usePickOnMap() {
  const router = useRouter();
  const { selectLocation, selectPlace } = useMapSelection();
  const pickLocation = useCallback((location: Location) => { selectLocation(location); router.push("/map"); }, [selectLocation, router]);
  const pickPlace = useCallback((place: Place) => { selectPlace(place); router.push("/map"); }, [selectPlace, router]);
  return { pickLocation, pickPlace };
}
```

### A client section with a colocated data hook

Nearby needs the browser's position, so it is a client section.

```tsx
// components/search/nearby-locations-data.ts
export type NearbyLocation = { location: Location; distanceMeters: number };

export function useNearbyLocations(): NearbyLocation[] {
  const position = useCurrentPosition();
  return useMemo(() => (position ? nearestLocations(position, LOCATIONS, 5) : []), [position]);
}

// components/search/nearby-locations.tsx
"use client";
export function NearbyLocations() {
  const nearby = useNearbyLocations();
  const { pickLocation } = usePickOnMap();

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

### A server section that loads its own data

Featured venues do not depend on the user, so the section fetches on the
server. Only the clickable card is a client component.

```tsx
// components/search/featured-locations-data.ts
export const getFeaturedLocations = cache(async () => db.location.findMany({ where: { featured: true } }));

// components/search/featured-locations.tsx (server component)
export async function FeaturedLocations() {
  const featured = await getFeaturedLocations();

  if (featured.length === 0) {
    return null;
  }

  return (
    <Section title="Featured">
      {featured.map((location) => <FeaturedLocationCard key={location.id} location={location} />)}
    </Section>
  );
}
```

### A section that takes one prop

```tsx
// components/search/place-results.tsx
"use client";
interface Props {
  query: string;
}

export function PlaceResults({ query }: Props) {
  const position = useCurrentPosition();
  const { places, searching } = usePlaceSearch(query, position);
  const { pickPlace } = usePickOnMap();

  if (searching) {
    return <Spinner />;
  }

  if (places.length === 0) {
    return <EmptyState title={`No results for "${query}"`} />;
  }

  return (
    <ul>
      {places.map((place) => (
        <PlaceResultRow key={place.id} place={place}
          distanceMeters={position ? distanceInMeters(position, place) : null}
          onPress={() => pickPlace(place)} />
      ))}
    </ul>
  );
}
```

### A pure leaf

```tsx
// components/search/location-result-row.tsx
interface Props {
  location: Location;
  distanceMeters: number | null;
  onPress: () => void;
}

export function LocationResultRow({ location, distanceMeters, onPress }: Props) { ... }
```

The row never imports the router, the store or a fetch. That is why the same
row is reused in the search list and in the map's side panel.

## Resulting layout

```
app/search/page.tsx                  page: reads ?q=, places sections
components/search/
  search-box.tsx                     client leaf, writes ?q=
  featured-locations.tsx             server section, no props
  featured-locations-data.ts         colocated loader
  featured-location-card.tsx         client leaf
  nearby-locations.tsx               client section, no props
  nearby-locations-data.ts           colocated data hook
  place-results.tsx                  client section, takes `query`
  place-result-row.tsx               leaf
  location-result-row.tsx            leaf
hooks/use-pick-on-map.ts             shared action hook
hooks/use-current-position.ts        shared data hook
utils/distance.ts                    nearestLocations, distanceInMeters (pure, reusable)
```
