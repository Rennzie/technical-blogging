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

With the COG hosted ready read go, all we need is a way to display it. While the COG data is organised so making range requests is possible, the client app has the responsibility of making those requests. A client uses the maps view port and resulting zoom level to make these requests and then renders the result. The [GeoTIFF](https://github.com/GeoTIFF) project maintained by [Daniel Dufour](https://github.com/DanielJDufour) has libraries to do both fetching and rendering. The [georaster](https://github.com/GeoTIFF/georaster) library provides an interface for making range requests and the [georaster-layer-for-leaflet](https://github.com/GeoTIFF/georaster-layer-for-leaflet) plugin works along side it to render the COG on a Leaflet map.

We use React which means there are a few extra hoops when using Leaflet. Fortunately [react-leaflet](https://react-leaflet.js.org/) has excellent bindings an it's recently been through a major release (v3). It's new core api makes building custom components much easier. The examples below will make use of `react-leaflet` v3.x. If you've not upgraded yet take a look at [this migration guide](https://sean-rennie.medium.com/migrating-react-leaflet-from-v2-to-v3-12d6088af191) written by Sean, alternatively take a look at [this Code Sandbox](https://codesandbox.io/s/react-leaflet-georaster-forked-tzl15) for an example using v2.x.

#### Writing a custom react-leaflet wrapper

The wrapper we created combines `georaster-layer-for-leaflet` and `georaster` into a single component. There are a few required dependencies to install for building the component.

```bash
yarn install georaster leaflet react-leaflet @react-leaflet/core georaster-layer-for-leaflet
```

`georaster-layer-for-leaflet` also has a dependency on [proj4](https://github.com/proj4js/proj4js#readme). It's [planned](https://github.com/GeoTIFF/georaster-layer-for-leaflet/issues/52) to allow injection, but for now it'll need to be added via a script tag.

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

`react-leaflets` core api provides high level factory functions for building Leaflet components. It abstracts away logic for adding and removing a layer from the map as well as adding and cleaning event listeners. There is a lot going on there but what's important is that `createPathComponent` allows the rendering of map elements by Leaflet while taking care of any issues React reconciler might cause like unnecessarily adding and removing a layer every render.

<!-- To be completely honest we're not 100% sure how these requests happen but it feels a bit like magic when the they appear in the network tab. -->

The only required option for `georaster-layer-for-leaflet` is `georasters` which takes an array of `Georaster` instances. It's this interface that's responsible making range requests to the COG file. When multiple `Georaster` instances are supplied it means we can render more than one COG simultaneously, effectively stacking layers in the client. The `useGeoraster`  hook wraps the pieces required to kick off requests by feeding one `parseGeoraster` call per path into `Promise.all()`. If successful it will return an array of `Georaster` objects. We've also included some status indicators as well for completeness.

#### Putting GeoRasterLayer to use

Right, with that out the way we can get onto the fun stuff. Our `GeoRasterLayer` takes three important props, `paths`, `pixelValuesToColorFn` and `resolution`, together they fetch and control how the image is rendered. `paths` is an array of urls to accessing the COG(s). To illustrate whats going on we'll only use one COG but the other props work in the same way if there are two or more.

The `resolution` determines how the sampling of pixels is done per tile. This [issues](https://github.com/GeoTIFF/georaster-layer-for-leaflet/issues/51) explains it better but whats worth noting is the higher the resolution the more intensive client CPU requirements are while render. The value depends on our users and we found `128` to a good middle ground between performance and image quality.

```javascript
const COG_PATH = "PATH_TO_COG";
const RESOLUTION = 128;

function App() {
  return (
    <MapContainer style={MAP_STYLES} center={position} zoom={19} maxZoom={25}>
      <GeoRasterLayer
        paths={[COG_PATH]}
        resolution={RESOLUTION}
        pixelValuesToColorFn={setPixelColours}
      />
    </MapContainer>
  );
}
```

To render an image on screen we'll apply a colour to every pixel. This is handled internally during the rendering of `canvas` elements but the `pixelValuesToColorFn` lets us control what colour gets applied to each pixel. The function is passed a `values` argument which is an array of bands values dependant on the COG. If the image is RGB the values will be an array of three bands: `[ R, G, B ]`. If it's a single band `NIR` image then the value will be an array with one value. When multiple images are used (by using multiple paths) the values argument combines both images. The value order depends on that of the urls supplied to `paths`, so if `paths={[ NIR_URL, RED_URL ]}` the resulting values argument would be `[ NIR, Red ]`. Ultimately `pixelValuesToColorFn` must return a valid [CSS colour](https://developer.mozilla.org/en-US/docs/Web/CSS/color) string.

How the function is used really depends on the underlying dataset. This example is for an RGB image:

```javascript
import chroma from "chroma-js";

function setPixelColours(values) {
  // Check if the pixel should be coloured, if not make it transparent
  if (!hasDataForAllBands(values)) {
    return "#00000000";
  }

  // destructure the bands
  const [red, green, blue] = values;
  const color = chroma(red, green, blue);

  return color;
}

const hasDataForAllBands = (values) =>
  values.every((value) => value != false || isNaN(value));
```

Firstly we check if the pixel should be coloured. Again, what needs to be checked in `hasDataForAllBands` will depend on the dataset but the general idea is to make pixels with no data values transparent. Our go to library for calculating colours from values is `chroma-js`. For an RGB colour all it needs is the three band values.

A more complex example like the one below calculates NDVI on the fly from a stacked multispectral COG. The layers could just have easily come from multiple COGs too.

```javascript
import chroma from "chroma-js";

function setPixelColours(values) {
  // No data check hidden for brevity

  const [_, __, red, nir] = values;
  const ndvi = calcNdvi(nir, red);
  const color = scale(ndvi).hex();

  return color;
}

function calcNdvi(nir, red) {
  return (nir - red) / (nir + red);
}

const scale = chroma.scale(["#B31C08", "#E6E60B", "#1FB308"]).domain([0, 0.7]);
```

This functionality makes it possible to produce many outputs on the fly from the same set of images. `georaster-layer-for-leaflet` even makes it possible to render custom canvas images for each pixel like this [wind direction example](https://geotiff.github.io/georaster-layer-for-leaflet-example/examples/wind-direction-arrows.html).
