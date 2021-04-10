# Cloud Optimised GeoTIFFs in production

Leverage Cloud Optimised GeoTIFFs (COGs) to display high res drone imagery on a web application. Save on processing and storage at the same time.

Written by:

- Alasdair Hitchins
- Sean Rennie

COGs leverage HTTP GET range requests making them a lower cost alternative to traditional map tiles. In our view they are equally as performant, with the advantage of being less power hungry during processing (up to 10x) and require less storage (savings of 40%-80%) for the equivalent resolution. They are also more versatile. COGs are simply GeoTIFFs and can be used for image processing in all the usual ways - they can replace your "original" TIFF too. Map tiles, on the other hand, are more specific to web applications. If you need to process that image again you'll have to store tiles and TIFFs.

More information about COGs can be found at their home: [cogeo.org](https://www.cogeo.org/).

The use of COG's at Hummingbird was born from the need to display high resolution drone imagery on our platform. We've been using map tiles, but to save on processing we'd only tiled them to zoom level 20. New products use imagery up to 5cm GSD for counting lettuces or picking out weeds in an empty field. These images resolve down to zoom level 25 so being the cost sensitive start-up we are COGs became an obvious choice.

Our go to is to generate the COGs with the [Geospatial Data Abstraction Library (GDAL)](https://gdal.org/). Support for direct COG generation was introduced in `v3.1` - our examples use `v3.2.2`. We first make Virtual Raster Tiles (VRTS) from multiple source GeoTIFFs: This step is optional but we do it because we store each wavelength of an image as individual band TIFFs. A COG is generated from the VRT and "hosted" in a Google Cloud bucket. Our React application is the primary user of the COGs. We leverage `georaster-layer-for-leaflet` and `georaster` javascript libraries to build a custom `react-leaflet` component for rendering.

---

## What is a COG and why should we care?

COGs are a specific configuration of the widely used GeoTIFF image format that are optimised by developers for storage on cloud services. They share the same file extension (`.tif`) as a regular TIFF or GeoTIFF file, so what's the difference?

### 1. Tiling

When storing regular TIFFs, the pixel data can be stored as blocks instead of line by line (A nice explanation [can be found here](https://kokoalberti.com/articles/geotiff-compression-optimization-guide/)). This allows 2-dimensional chunks of pixel data to be accessed without reading the entire dataset. Effectively you only need to work with the section of the image your require when reading or writing pixels. It makes accessing the dataset a lot more performant if you're working on large datasets and don't want to load everything in memory or download everything from a storage bucket. COGs make use of this feature to enable faster I/O operations for cloud resources, which we will see later.

### 2. Overviews

Another nice feature in GeoTIFFs is storing overviews - sometimes referred to as pyramid layers. They are lower resolution versions of the dataset embedded in the file itself. While this slightly increased the overall filesize, it makes visualisation of the dataset more performant when the desired output visual is lower resolution than the original dataset. Think of it being similar to viewing a thumbnail image or preview. You might have seen this when loading an image into your graphic GIS software or viewing a raster on a webmap canvas and increasing or decreasing the zoom level.

So what makes COGs different from regular GeoTIFFs? Well, they are regular GeoTIFFs but with tiling and overviews configured to make them best suited for webGIS applications and remote storage I/O operations.

## 1. Generating VRTs from separate rasters

GDAL enables raster datasets to be created virtually from other raster datasets without copying all of the pixel data. A VRT file is an XML format that references one or more image files and their associated metadata. This also allows you to interact with all of them as if they were one single multi-band/stacked TIFF.

## Why are we using VRTs to generate COGs?

Common practice in public earth observation datasets is to keep each image band of a given wavelength separate. For Sentinel-2 datasets you can see the configurations [here](https://sentinel.esa.int/web/sentinel/technical-guides/sentinel-2-msi/msi-instrument). We used this practice for storing our UAV orthomosaic data presenting us with two options; 1) COG each band and request the files individually, 2) or stack the bands and create a single COG. Making one request has it's benefits (less network requests for the client), so we chose to go with the latter.

The VRT allows us to stack the bands in the correct order.

## Creating a VRT

Creating a VRT is straight forward using the command [`gdalbuildvrt`](https://gdal.org/programs/gdalbuildvrt.html?highlight=gdalbuildvrt). If you follow the link you can see an example. They're very lightweight and provide a great interface to handling separate image files in the same context. Imagine we have three separate TIFFs: `red.tif`, `green.tif`, `blue.tif` corresponding to different bands of the same image. We can build a VRT to create a band-separated RGB image that is handled as a single file: `rgb.vrt`. We want to keep the image bands separate to ensure they don't get merged into a single band in the resulting VRT. This would result in a greyscale image - not what we want.

```bash
gdalbuildvrt -separate rgb.vrt red.tif green.tif blue.tif
```

This creates a VRT for an RGB dataset. We can now load the resultant `rgb.vrt` into our GIS software or have GDAL handle it as if it were a multi-band TIFF - like creating a "stacked" COG.

## 2. Creating COGs using GDAL

To create a COG using `GDAL>=3.1`, we will use the `gdalwarp` command. There are a few creation options (`-co`) that need to be specified, for our `react-leaflet` component to handle the COG correctly. Looking at the [COG driver page](https://gdal.org/drivers/raster/cog.html) from the GDAL documentation, we can see that it's essentially the TIFF driver that is built-in by default with GDAL plus some preprocessing already applied. Here are some basic creation options you might want to start with:

### 1. TILING_SCHEME

This option is used for overview and tiling generation. We use the built in `GoogleMapsCompatible` one which generates overviews that correspond to zoom levels on a Google Maps standard webmap canvas. See more about webmap projections, zoom levels and performance on the Google Design blog [here](https://medium.com/google-design/google-maps-cb0326d165f5).

### 2. COMPRESS

We also want to compress our file using lossless compression to reduce physical file size, without losing any information. In this case, we use `DEFLATE`. [Wikipedia](https://en.wikipedia.org/wiki/Deflate) gives a deeper dive on this algorithm and other options.

With this we've got all the basic information required to make a COG that is suitably performant for our Leaflet app:

```bash
gdalwarp -of COG -co TILING_SCHEME=GoogleMapsCompatible -co COMPRESS=DEFLATE rgb.vrt rgb-cog.tif
```

*If you didn't create a vrt you can replace `rgb.vrt` with your stacked tif - e.g. `rgb.tif`.*

## 3. Handling COG generation as part of a Python microservice

Creating COGs on the command line is all well and good, but we wanted this conversion to be performed within our data processing pipeline. Specifically after satellite data processing or UAV orthomosaic generation and quality assurance tests are complete. We need to handle the commands as part of a microservice that could be called whenever we have data we want to translate from a typical GeoTIFF to a COG and store it. These generalised code snippets show how we do this:

### 1. Handle the command execution

We want to ensure that the command runs, logging any exceptions that might occur in a logbook for debugging later:

```python
from subprocess import run, CalledProcessError, **CompletedProcess**
from shutil import which
from typing import List


def run_command(command: List, timeout: int=600) -> CompletedProcess:
    """Run a subprocess command with error handling and logging functionality.
    """
    process = run(command, capture_output=True, timeout=timeout)

    try:
        process.check_returncode()
        # Log successful command...
        return process

    except CalledProcessError as e:
        ...
        # Send (process.returncode, process.stderr) to logging.
        if which(command[0]) is None:
            raise RuntimeError(f"{command[0]} was not found, is it installed and on the path?")
        else:
            raise ...

```

### 2. Build the commands based on inputs and run

Next we want to create our commands and run them. We don't really want to keep the vrt file so we will create a temp file to store it. You can wrap your command generation to allow more flexible handling of the creation options that are sent to GDAL based on what images you are handling.

```python
from typing import List
from pathlib import Path
import tempfile


def run(input_paths: List[Path], output_cog_path: Path) -> None:

    temp_vrt_path = (Path(tempfile.gettempdir()) / next(tempfile._get_candidate_names())).with_suffix(".vrt")

    vrt_command = ["gdalbuildvrt", "-separate", temp_vrt_path, *input_paths]

    run_command(vrt_command)

    cog_command = ['gdalwarp', '-of', 'COG', '-co', 'TILING_SCHEME=GoogleMapsCompatible', '-co', 'COMPRESS=DEFLATE', temp_vrt_path, output_cog_path]

    run_command(cog_command)

    if not output_cog_path.exists():
        raise FileNotFoundError("The COG was not created, see the full logs for more details.")

```

We are assuming that the file paths and additional parameters have been sent to the service:

```python
inputs_paths = ["red.tif", "green.tif", "blue.tif"]

output_cog_path = "rgb.tif"

run(input_paths, output_cog_path)

# Send `output_cog_path` to the cloud storage bucket.
```

If this completes successfully, we can send our resultant COG back to the cloud storage bucket where it can be requested by the application.

## 4. Hosting COGs for consumption
<!-- Section by SR -->

Very simple, stick it in a bucket that allows range requests. We use Google Cloud which is configured for this by default. There's no need for a web server if you have the ability to make range requests from the client.

Not something we do, but it's possible to generate map tiles or Mapbox vector tiles on the fly from your COGs. Take a look at [rio-tiler](https://github.com/cogeotiff/rio-tiler) and [rio-tiler-mvt](https://github.com/cogeotiff/rio-tiler-mvt) if you're interested.

## 5. Consuming COGs with React & Leaflet
<!-- Section by SR -->

With the COG hosted and ready go, all we need is a way to display it. The COG data is organised so making range requests is possible and the client app is responsibility for making them. A client uses the maps view port and resulting zoom level to make these requests and then renders the result. The [GeoTIFF](https://github.com/GeoTIFF) project maintained by [Daniel Dufour](https://github.com/DanielJDufour) has libraries to do both fetching and rendering. The [georaster](https://github.com/GeoTIFF/georaster) library provides an interface for making range requests from the client and the [georaster-layer-for-leaflet](https://github.com/GeoTIFF/georaster-layer-for-leaflet) plugin uses it to render the COG on a Leaflet map canvas.

We use React which means there are a few extra hoops for using Leaflet. Fortunately [react-leaflet](https://react-leaflet.js.org/) has excellent bindings and it's recent major release (v3) comes with a core api for building custom components. The examples below make use of `react-leaflet@v3.x`. If needed, [this Code Sandbox](https://codesandbox.io/s/react-leaflet-georaster-forked-tzl15) uses 2.x or see [this migration guide](https://sean-rennie.medium.com/migrating-react-leaflet-from-v2-to-v3-12d6088af191) - written by Sean.

### Writing a custom react-leaflet wrapper

The wrapper we created combines `georaster-layer-for-leaflet` and `georaster` into a single component. There are a few required dependencies to build it.

```bash
yarn install georaster leaflet react-leaflet @react-leaflet/core georaster-layer-for-leaflet

||

npm add georaster leaflet react-leaflet @react-leaflet/core georaster-layer-for-leaflet
```

`georaster-layer-for-leaflet` also has a dependency on [proj4](https://github.com/proj4js/proj4js#readme). It's [planned](https://github.com/GeoTIFF/georaster-layer-for-leaflet/issues/52) to allow injection, but for now it'll need to be added via a script tag.

```html
  <!-- Pasted in the document head -->
  <script src="https://unpkg.com/proj4"></script>
```

And the custom component:

```javascript
// GeoRasterLayer.jsx

import * as React from "react";
import parseGeoraster from "georaster";
import GeoRasterLayerForLeaflet from "georaster-layer-for-leaflet";
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

`react-leaflet`'s core api provides high level factory functions for building Leaflet components. It abstracts away logic for adding and removing a layer from the map as well as adding and cleaning event listeners. There is a lot going on there but what's important is that `createPathComponent` allows the rendering of map elements by Leaflet while taking care of any issues React might cause - like unnecessarily adding and removing layers every render.

The only required option for `georaster-layer-for-leaflet` is `georasters` which takes an array of `Georaster` instances. It's this interface that's responsible for making range requests to the COG file. When multiple `Georaster` instances are supplied it means we can render more than one COG, think band stacking but in the client. We did this stacking during the COG creation process so won't need to use multiple files here. The `useGeoraster`  hook wraps the pieces required to kick off requests by feeding one `parseGeoraster` call per path into `Promise.all()`. If successful it will return an array of `Georaster` objects. We've also included some status indicators as well for completeness.

#### Putting GeoRasterLayer to use

Our `GeoRasterLayer` takes three important props, `paths`, `pixelValuesToColorFn` and `resolution`, together they fetch and control how the image is rendered. `paths` is an array of urls to the COG(s).

The `resolution` determines how the sampling of pixels is done per tile. This [issue](https://github.com/GeoTIFF/georaster-layer-for-leaflet/issues/51) explains it better but whats worth noting is the higher the resolution the more intensive client CPU requirements are while rendering. We're still figuring out the best value as it depends on our users. For now we are using `64` which looks like a good middle ground between performance and image quality. This value must be a power of 2.

```javascript
const COG_PATH = "PATH_TO_COG";
const RESOLUTION = 64;

function App() {
  return (
    <MapContainer style={ MAP_STYLES } center={ position } zoom={ 19 } maxZoom={ 25 }>
      <GeoRasterLayer
        paths={[ COG_PATH ]}
        resolution={ RESOLUTION }
        pixelValuesToColorFn={ setPixelColours }
      />
    </MapContainer>
  );
}
```

The `pixelValuesToColorFn` gives us control over which colour gets applied to each pixel when the `canvas` is rendered. It's passed a `values` parameter which is an array of band values - which will be dependant on our COG. Ours is RGB and values will be an array of three bands: `[ R, G, B ]`. If it's a single band `NIR` image then the value will be an array with one value. When multiple images are used (by using multiple paths) passed to `values` combines both and the order will depend on that of the urls supplied to `paths`. For example, if `paths={[ NIR_URL, RED_URL ]}` the resulting values argument would be `[ NIR, Red ]`. `pixelValuesToColorFn` must return a valid [CSS colour](https://developer.mozilla.org/en-US/docs/Web/CSS/color) string which gets applied to the canvas.

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

Firstly we check if the pixel should be coloured. Again, what needs to be checked in `hasDataForAllBands` will depend on the dataset.  The general idea is to make pixels with no data values transparent. Our go to library for calculating colours from values is `chroma-js`. For an RGB colour all it needs is the three band values.

A more complex example, like the one below, calculates NDVI on the fly from a stacked multispectral COG. The layers could also have come from multiple COGs.

```javascript
import chroma from "chroma-js";

function setPixelColours(values) {
  // No-data check hidden for brevity

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

## 6. Performance
<!-- Section by SR -->

The story for COGs at Hummingbird is about cost benefit. In our case, cloud compute and storage cost versus higher resolution images in our web platform. User experience is also key.

Diving into storage and processing performance we noticed map tiles become more expensive than COGs as soon as you tile over zl 20. As we've mentioned we need resolution down to zl 24 or 25 for our 5cm GSD products. Some back of the envelope calculations show we'd be saving up to 45% in storage costs by using COGs. There's another win there too, when its all said and done, you only need one remaining file - the COG. No need to store the tile set and the original GeoTIFF.

When it comes to processing, COGs are often up to 10x faster to generate than tiles. In our books this is a massive win.

Measuring on platform performance is a bit trickier and subjective. What is considered a "fast" load to a user navigating a lettuce field. Is that the most important metric or is seeing greater detail at lower zoom levels important? We are actively evaluating this question, but for the time being are happy with the perceived performance of COGs when compared to map tiles. If we get a more concrete answer we'll be sure to write up a new post with our learnings.
