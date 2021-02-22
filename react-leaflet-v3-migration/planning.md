# Writing Plan

## Authors

- Sean Rennie

## Title

- Migrating react-leaflet from v2 to v3

## Blurb

- Paul LeCam's `react-leaflet` library is a powerful set of react bindings for `leaflet`. Recently the library has been overhauled to modern react, making use of hooks and doing away completely with classes. It's also been re-written in typescript 🎉! In my view this work brings `react-leaflet` closer to a direct binding for leaflet rather than a "react flavoured leaflet". The introduction of a core library makes building custom components really easy giving react developers the full power of leaflet and its plugin ecosystem. Hats off to you Paul, thanks for the amazing work. Below is my attempt at outlining the changes our app needed in order to migrate to v3.

## Ideas

- Initial Assumptions
  - Using typescript
  - already using `useLeaflet` instead of  `withLeaflet`
- Summary of changes
  - classes => hooks
  - typescript as 1st class citizen
  - public and core API's
  - no map ref in `MapContainer`  🛑
  - More hooks (map & events)

- Who the changes effect
  - Apps with custom components
  - Custom wrappers out leaflet plugins
  - Apps that rely on maps refs outside of the map component

- Steps to success
  - Prepare the code base
    - Review all custom components, simplify what you can.
    - Make sure you are using leaflet effectively (GeoJson as geoJson etc, renderer)
  - Upgrade the dependencies
    - If typescript: remove `@types/react-leaflet`
  - Change `Map` to `MapContainer` updating props as required. This is a major break with a number of key differences
    - no `onViewPortChange`
    - *Look through the changes and add some more here*
    - No longer "controlled" props are immutable
  - update `useLeaflet` to `useMap`
  - GeoJSON with `pathOptions` & `eventHandler` instead of direct props. i.e: `renderer` `onClick`
  - Refactor Custom components with `@react-leaflet/core`

```jsx
  // 3.0
<MapContainer
  center={[51.505, -0.09]}
  zoom={13}
  className={classes.map}
  maxZoom={22}
  zoomControl={false}
  attributionControl
  preferCanvas
  renderer={L.canvas()}
  trackResize
  whenCreated={map => {
    //... do something with `map` ref
  }}
>
  // ... map layers
</MapContainer>

```

```jsx
// 2.0
<Map
  ref={mapRef}
  className={mapClassNames}
  viewport={viewport}
  // only use when this is the source map. Sync will take care of the comparison instance
  onViewportChanged={(event) => setViewport(event)}
  maxZoom={22}
  attributionControl={false}
  zoomControl={false}
>
//... map layers
</Map>
```

- Special cases
  - Google Maps
  - Syncing two maps

```typescript
type MapSyncProps = {
  otherMap: React.RefObject<L.Map>;
};

function MapSync({ otherMap }: MapSyncProps) {
  const map = useMapEvents({
    drag: ({ target }) => otherMap.current && sync(otherMap.current, target),
    zoomend: ({ target }) => otherMap.current && sync(otherMap.current, target),
  });

  // note: Effect ensure maps are synced on first load - SFR 2021-01-28
  React.useEffect(() => {
    if (otherMap.current) {
      sync(otherMap.current, map);
    }
  }, [map, otherMap]);

  return null;
}
```

## Open Questions
