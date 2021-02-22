# Migrating react-leaflet from v2 to v3

*[Paul LeCam's](https://github.com/PaulLeCam) [react-leaflet](https://react-leaflet.js.org/) library is a powerful set of react bindings for [Leaflet](https://leafletjs.com/index.html). Recently the library has been overhauled to modern react, making use of hooks and doing away completely with classes. It's also been re-written in typescript 🎉! In my view this work makes `react-leaflet` more of a direct binding to `leaflet` rather than a "react flavoured" of it. The introduction of a core library makes building custom components really easy. It gives react developers the full power of leaflet and its [plugin ecosystem](https://leafletjs.com/plugins.html).*

*Hats off to you Paul, thanks for the amazing work. Below is my attempt to add something to it by outlining the changes our app needed in order to migrate to v3.*

## Some Initial Assumptions

This guide is written for those already comfortable with `leaflet` and `react-leaflet`. The examples are written in Typescript where applicable but shouldn't be a blocker for Javascript migrations. Your application is written with or migrating to react hooks and functional components.

## So whats new in V3?

### These break things

- `Map` component renamed to `MapContainer`
  - Numerous prop changes between the two as well, i.e: no more `viewPort` or `onViewPortChanged`
  - Most of the props are immutable after first load
- No more `useLeaflet` hook. Its been replaced with `useMap` instead.
- `MapContainer` no longer has a ref.
- A number of props in various components. I won't list them here as we mostly use the GeoJson component and only had to update any `renderer` and event props with `pathOptions` and `eventHandler` respectively

### New and Shiny

- `useMapEvents` and `useMapEvent` hooks. They allow you to register map events on a component.
- Typescript, Typescript, Typescript. This is good news even if you don't use TS. The intellisense and autocomplete is even better than before and is guaranteed to stay up to date.
- A core API at `@react-leaflet/core`. You'll only use this if you are building custom components. Its particularly useful when you need a leaflet plugin that doesn't have a react-leaflet wrapper.

### And these have been removed

- Class based components. No more `createLeafletElement` 🎉. Everything is built on hooks and its better for it!

## Everyone should think about updating

Everyone should consider upgrading to version 3 mostly because v2 is no longer maintained but also because (IMHO) version 3 is much closer to using leaflet than version 2 was. I found myself reaching for the leaflet docs way more. I also feel like I've now got a much better grasp on the inner working of leaflet and how to manipulate it in react. The truth is, because leaflet controls the DOM (every react leaflet component ultimately renders `null` in react land), approaching things with a react first approach pushed me in the wrong direction. Version 3 brings those principles home in a great way.

If your use leaflet to render a simple map with a tile layer or two then the changes will be minimal. If you have a host of custom components or any special cases (like syncing two maps) then the migration will be a little more involved. If you've been wanting to use leaflet but can't because your case requires a leaflet plugin that doesn't have a react-wrapper, now is the perfect time to dive in. Building custom components is a real treat in version 3.

## You get it done by following the errors

//? It may be obvious but simply follow the errors in the console. It's a sure fire way to work through everything that needs to change. That being said, not everything showed up in the console. Fortunately our project is also written in typescript so leaning on the type-checker got the rest of the issues sorted.

  *Below I'll illustrate the bits that needed to change to upgrade to V3. Following the errors in the console as well as the type errors from the compiler lead me through all the changes I had to make. Roughly speaking I did things in the order written but this may not be the case for you.*

### Pre Game

*Re-write any components making use of `useLeaflet` hook to functions*

It really helped me to read through the documentation and see what sort of changes I'd need to make. It was also a great opportunity to assess how we used leaflet and clean up components and places where we hadn't used it very well. A common example goes something like this:

```javascript
foo.map(geometry => <GeoJSON data={geometry}/>)
```

This is a very common pattern in react but is unnecessary in leaflet. GeoJSON is designed to take multiple geometries at once in a FeatureCollection. It's better to transform your data and simply render one component:

```javascript
<GeoJSON data={fooCollection}/>
```

I realise this doesn't have much to do with upgrading to version 3, but now is as good a time as any to clean out those bad practices.

### The basics

Upgrade `react-leaflet` with your favourite package manager. If you use typescript, remove `@types/react-leaflet`. Version 3 has them built in so you want to make sure the compiler is using the correct ones.

### Giving over control to MapContainer

A common pattern in react is to use controlled components. To keep track of the maps `viewport` (center & zoom) a developer could do something like:

```javascript
import {Map} from 'react-leaflet@2.x'

function MapParentWrapper(){
  const [viewport, setViewport] = useState(defaultViewport)

  const [zoomControl, setZoomControl] = useState(false)
  const [showMapAttribution, setShowMapAttribution] = useState(false)
  return (
  <Map
    viewport={viewport}
    onViewPortChange={setViewport}

    // other dynamic props
    maxZoom={22}
    attributionControl={showMapAttribution}
    zoomControl={zoomControl}
  >
  {/** your map components */}
  </Map>
  )
}
```

Then pass `setViewport` to any child component that needs to make updates programmatically. Again this is a common pattern in react, update a prop, component "reacts". But its key to remember that we are working with leaflet which controls the DOM so we should really do things the leaflet way.

V3 gets rid of the controllable `Map` component and replaces it with `MapContainer`. The key difference here is that the `MapContainer` props are immutable after the initial load (The exception being `children`). You can set the start position of your map here along with any other options you need.  To access any internal map state you need to get hold of it's instance and go from there.

```javascript
import {MapContainer} from 'react-leaflet@3.x'

const LondonCoords = [51.505, -0.09];

function MapParentWrapper(){
  return (
    <MapContainer
      // set the initial position of the map
      center={LondonCoords}
      zoom={13}

      // set some options
      className={{/** don't forget an explicit height */}}
      maxZoom={22}
      zoomControl={false}
      preferCanvas

      // get access to the map instance
      whenCreated={map => {
        // do whatever makes sense. I've saved it as a ref
        }}
     >
       {/** your map components */}
    </MapContainer>
  )
}
```

Another key difference is that `MapContainer` no longer has a `ref` attribute. Paul indicates its [been problematic](https://github.com/PaulLeCam/react-leaflet/issues/806) so he's removed it as a mechanism to get an instance of the map. Reading through the source code, this is likely because V2 provided a ref as a map instance (which is what you might expect) but internally that ref is better served to link leaflet to the root div of `MapContainer` to do leaflet things. Don't worry though, Paul's made sure that if you need access to a map instance there are a few options.  You could use any of the new hooks (see below), or if you need access to it outside of `MapContainer` grab a reference using the `whenCreated` prop. There are a few other issues on github [here](https://github.com/PaulLeCam/react-leaflet/issues/841) and [here](https://github.com/PaulLeCam/react-leaflet/issues/846) with examples of ways to do just that.

### Hooks galore

In V2 you could access a map instance and various other pieces of context with the `useLeaflet` hook or the `withLeaflet` HOC. In V3 both of these have been removed and replace with `useMap`, `useMapEvent` and `useMapEvents`. Each hook returns the map instance and naturally you can only use these hooks in components below `MapContainer`.

Search your codebase for the `useLeaflet` hook it with `useMap`. Don't do a find a replace here. The `useMap` hooks only returns an instance of the map, it doesn't include the other properties `pane`, `layerContainer` & `popupContainer` that the old context did.

```javascript
// Your code should change from this:
  const { map } = useLeaflet()
// to this:
  const map = useMap()
```

If you needed any of the other properties, get them by calling the appropriate method on the map instance:

```javascript
 const pane = map.getPane('mapPane')
 const layerContainer = map.getContainer()
 // this should work but have not tested it myself
 const popupContainer = map.getPane('markerPane')?.getRootNode()
```

#### Events

V3 introduces to two new convenience hooks for attaching leaflet event listeners to your components. The `useMapEvent` and `useMapEvents` are used to attach an event or multiple events respectively. I'm wont go into details here as there is an example below in the special cases [section](//link-to-may-sync). Be sure to check out the [docs](https://react-leaflet.js.org/docs/api-map#usemapevents) for other examples to.

### Prop Changes

If you don't have any custom `react-leaflet` components you are in the home stretch now. Most of the library components have had changes to there props. This is probably because Paul has made far better use of the native leaflet class options to be the props for the relevant component. This is where I leaned heavily on the `tsc` compiler to help me out. I ran it in watch mode `tsc --watch` and worked my way through all the type errors. I my case our app exclusively uses `GeoJSON` and `TileLayer`. The latter needed no changes but for each instance of `GeoJSON` I needed to move all event handlers like `onClick` into the `eventHandler` prop and move the `renderer` prop into `pathOptions`. I typical case went from this:

```javascript
<GeoJSON
  data={fooGeoJson}
  render={L.canvas()}
  style={() => getGeoJSONStyle()}
  onClick={handleLayerClick}
/>

// to this:

<GeoJSON
  data={fooGeoJson}
  pathOptions={{ renderer: L.canvas() }}
  style={() => getGeoJSONStyle()}
  eventHandler={{
    onclick: event => handleLayerClick(event)
  }}
/>

```

This isn't documented and the TS complier will shout at you, but I found out that the `pathOptions` prop will accept a function which is passed a feature as an argument. This is similar to how `onEachFeature` works accept `layer` is not passed as a second arg. I raised [this issue](https://github.com/PaulLeCam/react-leaflet/issues/851) at time of writing to look into it.

There are likely a number of other prop differences between V2 and V3 but these are all I encountered.

### Syncing two maps. A special case

The biggest challenge I faced in the migration came from a special case. We need to show two, almost identical, map side by side. These maps are synced, if you pan or zoom one, the other will pan or zoom too. The maps need to be the same size and one could not sit inside the other one. To do this you need access to the map instance of each map. But, and this is key, only sync the maps when you are in comparison mode. In V2 we achieved this using [leaflet.sync](https://github.com/jieter/Leaflet.Sync). The library served us well but its been unmaintained for 3 years and it gave me headaches during the upgrade. Eventually I settled on building a custom `MapSync` component to handle the behaviour we wanted. This is what it looks like:

```typescript

type MapSyncProps = {
  otherMap: React.RefObject<L.Map>;
};

function MapSync({ otherMap }: MapSyncProps) {
  const map = useMapEvents({
    drag: ({ target }) => otherMap.current && sync(otherMap.current, target),
    zoomend: ({ target }) => otherMap.current && sync(otherMap.current, target),
  });

  // ensures the maps are synced when component mounts
  React.useEffect(() => {
    if (otherMap.current) {
      sync(otherMap.current, map);
    }
  }, [map, otherMap]);

  return null;
}

// a helper function
function sync(map1: L.Map, map2: L.Map) {
  map1?.setView(map2.getCenter(), map2.getZoom(), {
    animate: false,
    duration: 1,
  });
}

```

This component is intended to be rendered by each map. It's passed the instance of the `otherMap` and using the `useMapEvents` hooks, listens for `drag` and `zoom` events. When the events are trigged the `otherMap` can be synced. Lastly I've used a `useEffect` to sync the maps when `MapSync` mounts so that both maps start off synced.

I then use this component like this:

```javascript
const INITIAL_POSITION = [51.505, -0.09]
const INITIAL_ZOOM = 13;

function Maps({ children, isComparing = false }: Props): ReactElement {
  const mapLayers: any[] = Children.toArray(children).filter(Boolean);
  const firstMapRef = useRef<L.Map>(null!);
  const secondMapRef = useRef<L.Map>(null!);

  // note: Get functions handle crash when a user refreshes while in compare mode - SFR 2021-02-04
  const getSecondMapInitialCenter = () => {
    return firstMapRef.current
      ? firstMapRef.current?.getCenter()
      : INITIAL_POSITION;
  };
  const getSecondMapInitialZoom = () => {
    return firstMapRef.current ? firstMapRef.current?.getZoom() : INITIAL_ZOOM;
  };

  return (
    <section className="root">
      {mapLayers.map((childLayer, index) => (
        <div
          key={index === 0 ? 'first-map' : 'second-map'}
          className={classNames("map-container", {
            // styling that makes each map equal in size sitting side by side
            'map-half': Boolean(mapLayers.length > 1),
          })}
        >
          <MapContainer
            // give the map an explicit height
            className="map"
            center={
              index === 0 ? INITIAL_POSITION : getSecondMapInitialCenter()
            }
            zoom={index === 0 ? INITIAL_ZOOM : getSecondMapInitialZoom()}
            // keep a reference of each map. 
            whenCreated={map => {
              if (index === 0) {
                firstMapRef.current = map;
              }
              if (index === 1) {
                secondMapRef.current = map;
              }
            }}
          >
            {isComparing && (
              // only load MapSync when in comparison mode
              <MapSync otherMap={index === 0 ? secondMapRef : firstMapRef} />
            )}
            {childLayer}
          </MapContainer>
        </div>
      ))}
    </section>
  );
}
```

Lets walk through whats happening. The `Maps` component is passed either one or two sets of map layers. In our use case they are identical except for one or two minor things that don't make a difference in this example. I use `React.toArray` to create an array of the two children so I can map out two `MapContainers` when necessary. I've got two refs, one for each map and some helper functions to set the initail position and zoom. Remember, all props to `MapContainer` are immutable except for `children`. The helper functions ensure that the correct center and zoom are used when the maps mount, they also ensure that the page does not crash if a user refreshes. Usually we show only one map and on specific occasions we compare. It's easy to control what the seconds map position should be when the user enters compare mode, it's what ever the values of the first map are. But if a user refreshes the page while in compare mode? The second map would expect the first map to exists (which is might not yet) and then crash. If a page has refreshed we can safely assume both maps should use the initial values. That all that going on there.

I use the `whenCreated` prop to save a copy of each map instance as a ref. Then when we are in comparison mode we can render `MapSync` in each map passing it the ref of the other map. `MapSync` takes care of position the opposing map when a user interacts with either map.

I'm really happy with this outcome. The `useMapEvents` hook and the `whenCreated` prop really helped me out here. I could also get rid of an unmaintained library keeping maintenance overhead in our app down.

## Conclusion

I've loved migrating to `react-leaflet` V3. I think its an amazing update from the previous version and gives me a lot of control over leaflet in my react project.

I hoped you learned something or it helped you out of a tight spot. I'd love to hear any comments (good or bad) in the comments below.

Happy Coding.
