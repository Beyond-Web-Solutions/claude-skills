# Worked example: a search screen

From an Expo Router app with `@expo/ui` (SwiftUI on iOS, Jetpack Compose on
Android). The screen shows featured venues, the nearest pick-up points, and
place search results while typing. Picking anything selects it on the map and
closes search.

## Before: the route collects everything

```tsx
// app/search.tsx
export default function SearchScreen() {
  const [query, setQuery] = useState("");
  const { selectPlace, selectLocation } = useMapSelection();
  const userPosition = useCurrentPosition();
  const { places, searching } = usePlaceSearch(query, userPosition);
  const nearby = useMemo(
    () => (userPosition ? nearestLocations(userPosition, LOCATIONS, 5).map((e) => e.location) : []),
    [userPosition],
  );
  const onSelectPlace = useCallback((place: Place) => { selectPlace(place); router.back(); }, [selectPlace]);
  const onSelectLocation = useCallback((location: Location) => { selectLocation(location); router.back(); }, [selectLocation]);

  return (
    <>
      <Stack.Screen options={{ headerSearchBarOptions: { onChangeText: ({ nativeEvent }) => setQuery(nativeEvent.text) } }} />
      <Search
        query={query}
        places={places}
        searching={searching}
        featured={FEATURED_LOCATIONS}
        nearby={nearby}
        userPosition={userPosition}
        onSelectPlace={onSelectPlace}
        onSelectLocation={onSelectLocation}
      />
    </>
  );
}
```

`Search` then had an iOS and an Android file, each destructuring eight props,
each computing distances for rows, each wiring the same callbacks into rows.
Adding a section meant touching the route, the props type, and both screens.

Problems, in the skill's terms:
- The route is a container for three unrelated things (featured, nearby,
  place search). It has three reasons to change.
- `onSelectLocation` is drilled: route → screen → section → row.
- `userPosition` is drilled only so rows can show a distance.
- Both platform screens duplicate the distance maths and the wiring.

## After: route and screen compose, sections own their data

```tsx
// app/search.tsx — one piece of screen-level state, one prop
export default function SearchScreen() {
  const [query, setQuery] = useState("");
  return (
    <>
      <Stack.Screen options={{ headerSearchBarOptions: { placeholder: "Search for a place", autoFocus: true, onChangeText: ({ nativeEvent }) => setQuery(nativeEvent.text) } }} />
      <Search query={query} />
    </>
  );
}

// screens/search.tsx — default variant and the contract
export interface Props {
  query: string;
}

export function Search({ query }: Props) {
  return null;
}

// screens/search.ios.tsx
import type { Props } from "./search";

export function Search({ query }: Props) {
  return (
    <Host seedColor={BRAND_COLOR} style={{ flex: 1 }}>
      {query ? <PlaceResults query={query} /> : (
        <List>
          <FeaturedLocations />
          <NearbyLocations />
        </List>
      )}
    </Host>
  );
}

// screens/search.android.tsx — same sections, Material layout
import type { Props } from "./search";

export function Search({ query }: Props) {
  return (
    <Host seedColor={BRAND_COLOR} style={{ flex: 1 }}>
      <Surface modifiers={[fillMaxSize()]}>
        <Column modifiers={[fillMaxSize(), verticalScroll(), imePadding()]}>
          {query ? <PlaceResults query={query} /> : (<><FeaturedLocations /><NearbyLocations /></>)}
        </Column>
      </Surface>
    </Host>
  );
}
```

### Shared action hook

```tsx
// hooks/use-pick-on-map.ts
export function usePickOnMap() {
  const { selectLocation, selectPlace } = useMapSelection();
  const pickLocation = useCallback((location: Location) => { selectLocation(location); router.back(); }, [selectLocation]);
  const pickPlace = useCallback((place: Place) => { selectPlace(place); router.back(); }, [selectPlace]);
  return { pickLocation, pickPlace };
}
```

### A section with a colocated data hook

```tsx
// components/search/nearby-locations-data.ts — shared by both platform files
export type NearbyLocation = { location: Location; distanceMeters: number };

export function useNearbyLocations(): NearbyLocation[] {
  const position = useCurrentPosition();
  return useMemo(() => (position ? nearestLocations(position, LOCATIONS, 5) : []), [position]);
}

// components/search/nearby-locations.tsx — default variant and the contract
export function NearbyLocations() { return null; }

// components/search/nearby-locations.ios.tsx
export function NearbyLocations() {
  const nearby = useNearbyLocations();
  const { pickLocation } = usePickOnMap();

  if (nearby.length === 0) {
    return null;
  }

  return (
    <Section title="Nearby">
      <List.ForEach data={nearby} keyExtractor={(item) => item.location.id}>
        {({ item }) => (
          <LocationResultRow location={item.location} distanceMeters={item.distanceMeters}
            onPress={() => pickLocation(item.location)} />
        )}
      </List.ForEach>
    </Section>
  );
}

// components/search/nearby-locations.android.tsx
export function NearbyLocations() {
  const nearby = useNearbyLocations();
  const { pickLocation } = usePickOnMap();

  if (nearby.length === 0) {
    return null;
  }

  return (
    <Column>
      <Text style={{ typography: "titleMedium" }} modifiers={[padding(16, 24, 16, 8)]}>Nearby</Text>
      {nearby.map(({ location, distanceMeters }) => (
        <LocationResultRow key={location.id} location={location} distanceMeters={distanceMeters}
          onPress={() => pickLocation(location)} />
      ))}
    </Column>
  );
}
```

### A section that takes one prop

```tsx
// components/search/place-results.ios.tsx
import type { Props } from "./place-results";

export function PlaceResults({ query }: Props) {
  const position = useCurrentPosition();
  const { places, searching } = usePlaceSearch(query, position);
  const { pickPlace } = usePickOnMap();

  if (searching) {
    return <ProgressView />;
  }

  if (places.length === 0) {
    return <ContentUnavailableView title={`No results for "${query}"`} systemImage="magnifyingglass" />;
  }

  return (
    <List>
      <List.ForEach data={places} keyExtractor={(item) => item.id}>
        {({ item }) => <PlaceResultRow place={item} distanceMeters={position ? distanceInMeters(position, item) : null} onPress={() => pickPlace(item)} />}
      </List.ForEach>
    </List>
  );
}
```

### A pure leaf

```tsx
// components/search/location-result-row.tsx — the contract both platforms implement
export interface Props {
  location: Location;
  distanceMeters: number | null;
  onPress: () => void;
}

// location-result-row.ios.tsx / .android.tsx
import type { Props } from "./location-result-row";
```

The row never imports the router, the store or a fetch. That is why the same
row is reused in the search list and in the map's bottom sheet.

## Resulting layout

```
app/search.tsx                       route: query state → <Search query />
screens/search.{tsx,ios.tsx,android.tsx}   screen: places sections
components/search/
  featured-locations.{tsx,ios.tsx,android.tsx}   section, no props
  featured-location-card.{tsx,ios.tsx,android.tsx} leaf
  featured-location-card-size.ts     constants shared by platform files
  nearby-locations.{tsx,ios.tsx,android.tsx}     section, no props
  nearby-locations-data.ts           colocated data hook
  place-results.{tsx,ios.tsx,android.tsx}        section, takes `query`
  place-result-row.{tsx,ios.tsx,android.tsx}     leaf
  location-result-row.{tsx,ios.tsx,android.tsx}  leaf
hooks/use-pick-on-map.ts             shared action hook
hooks/use-current-position.ts        shared data hook
```
