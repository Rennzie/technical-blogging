# Comsuming COGs in a React Application

- Written By: Sean Rennie

- Plan:
  - COG is ready to fetch from the bucket
  - Don't walk through the whole custom component process
    - Show the end code and then talk it through
  - Intro libraries and explain what they are for
    - georaster
    - georaster-layer-for-leaflet
    - proj4
  - Style functions, RGB, MS or single band
  - Resolution

With the COG ready go, all we need is a way to display it. While the COG data is organised so making range requests is possible, the client app has the responsibility of making those requests. A client uses the maps view port and resulting zoom level to create the request effectively fetching whichever overlay and tiles are relevant. To achieve this we need two things, a way to render our image and a way to fetch relevant pieces of that image. The [GeoTIFF](https://github.com/GeoTIFF) project maintained by [Daniel Dufour](https://github.com/DanielJDufour) has libraries to do both. [georaster](https://github.com/GeoTIFF/georaster) library provides an interface for making range requests and the [georaster-layer-for-leaflet]() plugin works along side it to render the COG on a Leaflet map.

We use React so are a few extra hoops when using Leaflet. Fortunately the [react-leaflet](https://react-leaflet.js.org/) has excellent bindings an it's recently been through a major release (v3) and now has a core api making building custom components much easier. The examples below will make use of `react-leaflet` v3, if you've not upgraded yet take a look at [this migration guide](https://sean-rennie.medium.com/migrating-react-leaflet-from-v2-to-v3-12d6088af191) written by Sean. Alternatively take a look at [this Code Sandbox](https://codesandbox.io/s/react-leaflet-georaster-forked-tzl15) for an example using `react-leaflet` v2.x.

#### Writing a custom react-leaflet wrapper

The wrapper we created combines `georaster-layer-for-leaflet` and `georaster` into a single component. There are a few required dependencies to install for building the component.

```bash
yarn install georaster leaflet react-leaflet @react-leaflet/core georaster-layer-for-leaflet
```

`georaster-layer-for-leaflet` also has a dependency on [proj4](). It's [planned](https://github.com/GeoTIFF/georaster-layer-for-leaflet/issues/52) to allow injection, but for now it'll need to be a script tag.

```html
  <!-- Past in the document head -->
  <script src="https://unpkg.com/proj4"></script>
```

And the custom component:

```javascript
// GeoRasterLayer.jsx

import * as React from "react";
import parseGeoraster from "georaster";
import GeoRasterLayerForLeaflet from "./georaster-layer-for-leaflet";
import { createPathComponent } from "@react-leaflet/core";

const GeoRasterComponent = createPathComponent((options, context) => ({
  instance: new GeoRasterLayerForLeaflet(options),
  context
}));

const useGeoraster = (paths) => {
  const [status, setStatus] = React.useState("idle");
  const [georasters, setGeoraster] = React.useState(null);

  React.useEffect(() => {
    setStatus("loading");
    const promises = paths.map((path) => parseGeoraster(path));
    Promise.all([...promises])
      .then((georaster) => {
        setGeoraster(georaster);
        setStatus("success");
      })
      .catch(() => {
        setStatus("error");
      });
  }, [paths]);

  return { status, georasters };
};

export default function GeoRasterLayer({ paths, ...options }) {
  const { georasters, status } = useGeoraster(paths);

  return status === "success" ? (
    <GeoRasterComponent {...options} georasters={georasters} />
  ) : null;
}
```

`react-leaflets` core api provides high level factory functions for building various types of Leaflet components. `georaster-layer-for-leaflet` extends `GridLayer` so `createPathComponent`  will work the best. It abstracts away logic for adding and removing a layer from the map as well as adding and cleaning event listeners. There is a lot going on there but what's important is that `createPathComponent` allows the rendering of map elements by Leaflet while taking care of any issues React reconciler might cause like unnecessarily adding and removing a layer every render.

The options for `georaster-layer-for-leaflet` will be covered in detail below but whats important to know is that it accepts one or more `Georaster` objects. The `Georaster` object is used by `GeoRasterComponent` for making range requests. To be completely honest we're not 100% sure what goes on there but it feels a bit like magic when the requests appear in the network tab. What it does mean though, is that we can render more than one COG simultaneously, effectively stacking layers. This is useful of you've got multispectral data stored as different layers and the goal is a false colour image. `useGeoraster` accepts an array of paths which feed into `parseGeoraster` then and `Promise.all()`. This will initiate the request for both layers returning an array of `Georaster` object. We've included some status indicators as well for completeness.

#### Putting GeoRasterLayer to use

Right, with that out the way we can get onto the fun stuff. The `GeorasterLayerComponent` takes three important props, `path`, `pixelValuesTocolorFn` and `resolution` allowing control over how our tiff is rendered. `path` is the url to where the COG is stored

```javascript
function App() {
  return (
    <MapContainer style={MAP_STYLES} center={position} zoom={19} maxZoom={25}>
      <GeoRasterLayer
        path={LETTUCE_RGB_COG}
        resolution={128}
        pixelValuesToColorFn={setPixelColours}
      />
    </MapContainer>
  );
}
```
