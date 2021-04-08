# Cloud Optimised GeoTIFFs in production

Leverage Cloud Optimised GeoTIFFs (COGs) to display high res drone imagery on a web application. Save on processing and storage at the same time.

Written by:

- Alasdair Hitchins
- Sean Rennie

COGs leverage HTTP GET range requests making them a lower cost alternative to traditional map tiles. In our view they are equally as performant, with the advantage of being less power hungry during processing (up to 10x) and require less storage (savings of 40%-80%) for the equivalent resolution. They are also more versatile. COGs are simply GeoTIFFs and can be used for image processing in all the usual ways, and can replace your "original" TIFF too. Map tiles, on the other hand, are more specific to web applications. If you need to process that image again you'll have to store tiles and TIFFs.

More information about COGs can be found at their home: [cogeo.org](https://www.cogeo.org/), however we will explain some of the benefits and considerations we took in this article.

The use of COG's at Hummingbird was born from the need to display high resolution drone imagery on our platform. We've been using map tiles, but to save on processing we'd only tiled them to zoom level 20. New products use imagery up to 5cm GSD for counting lettuces or picking out weeds in an empty field. These images resolve down to zoom level 25 so being the cost sensitive start-up we are COGs became an obvious choice.

Our go to is to generate the COGs with the [Geospatial Data Abstraction Library (GDAL)](https://gdal.org/). We first make Virtual Raster Tiles (VRTS) from multiple source GeoTIFFs: This opiontal step is done when dealing with bands stored as individual TIFFs that correspond to different wavelengths as the same image with the same acquisition time. A COG is generated from the VRT and "hosted" in a Google Cloud bucket. Our React application is the primary user of the COGs. We leverage `georaster-layer-for-leaflet` and `georaster` javascript libraries to build a custom `react-leaflet` component for displaying the COGs.

Support for direct COG generation was introduced in GDAL `3.1`, however in this article we are using the latest version at the time of writing, `3.2.2`.

## What is a COG and why should we care?

Cloud Optimised GeoTIFFs (COGs) are a specific configuration of the widely used GeoTIFF image format that optimise images for storage on cloud services. They share the same file extension (`.tif`) as a regular TIFF or GeoTIFF file, so what's the difference?

#### 1. Tiling

When storing regular TIFFs, the pixel data can be stored as blocks instead of line by line (A nice explanation [can be found here](https://kokoalberti.com/articles/geotiff-compression-optimization-guide/)). This allows 2-dimensional chunks of pixel data to be accessed without reading the entire dataset, which means when you are reading or writing pixel data in a dataset, you are only working with the section of the image that you require. This makes accessing the dataset a lot more performant if you're working on large datasets and don't want to load everything in memory or download everything from a storage bucket. COGs make use of this feature to enable faster I/O operations for cloud resources, which we will see later.

#### 2. Overviews

Another nice feature in GeoTIFFs is storing overviews (sometimes referred to as pyramid layers), which are lower resolution versions of the dataset embedded in the file itself. While this slightly increased the overall filesize, it makes visualisation of the dataset more performant when the desired output visual is lower resolution than the original dataset, similar to vieiwing a thumbnail image or preview. You might have seen this when loading an image into your graphic GIS software or viewing a raster on a webmap canvas and increasing or decreasing the zoom level.

So what makes COGs different from regular GeoTIFFs? Well, they are regular GeoTIFFs but with tiling and overviews configured to make them best suited for webGIS applications and remote storage I/O operations.

## 1. Generating VRTs from separate rasters

One of the many features of GDAL enables raster datasets to be created virutally from other raster datasets without copying all of the pixel data. A VRT file is an XML format that references one or more raster datasets and associated their metadata to describe where each of them are positioned geographically. This also allows you to interact with all the datasets as if they were one single multi-band/stacked TIFF.

## Why are we using VRTs to generate COGs?

In our use case, we had followed the same structure for our UAV orthomosaic data storage that many public earth observation datasets have followed, which keeps each image band of a given wavelength separate. In the case of Sentinel-2 data, you can see the configurations [here](https://sentinel.esa.int/web/sentinel/technical-guides/sentinel-2-msi/msi-instrument). We could keep the same structure for our COGs too, however making one request to a single file is more beneficial than requesting a series files individually, so we chose to stack the bands into a singe image file.

In order to stack the bands into a multi-band image file, we will generate a VRT to order the bands correcty, and generate a single COG from the VRT, referencing the three separate images.

If you prefer to keep the bands separate, then this step is not necessary to generate COGs.

## Creating a VRT

Creating a VRT is pretty straight forward using the command [`gdalbuildvrt`](https://gdal.org/programs/gdalbuildvrt.html?highlight=gdalbuildvrt). If you follow the link you can see an example of a VRT file - they're very lightweight and provide a great interface to handling separate image files in the same context. Imagine we have three separate TIFFs: `red.tif`, `green.tif`, `blue.tif` corresponding to different bands of the same image. We can build a simple VRT to create a band-separated RGB image that is handled as a single file: `rgb.vrt`. We want to keep the image bands separate to ensure that they don't get merged into a single band in the dataset.

Here is the command:

```bash
gdalbuildvrt -separate rgb.vrt red.tif green.tif blue.tif
```

This creates a simple VRT for an RGB dataset, and we can now load the resultant `rgb.vrt` into our GIS software, or have GDAL handle it as if it were a multi-band TIFF.

## 2. Creating COGs using GDAL

To create a COG using `GDAL>=3.1`, we will use the `gdalwarp` command. There are a few creation options (`-co`) that need to be specified,for our react-leaflet component to handle the COG correctly. Looking at the [COG driver page](https://gdal.org/drivers/raster/cog.html) from the GDAL documentation, we can see that it's essentially the TIFF driver that is build-in by default with GDAL with some preprocessing already applied. We will cover some basic creation options you might want to start with here:

#### 1. TILING_SCHEME
Looking at the creation option `TILING_SCHEME`, we can see that there is a built in `GoogleMapsCompatible` option for overviews to be generated at the correct zoom levels that will correspond to incremental zoom levels on a webmap canvas and generate the correct overviews for us. You can read more information about webmap projections, zoom levels and performance on the Google Design blog [here](https://medium.com/google-design/google-maps-cb0326d165f5).

#### 2. COMPRESS
We also want to compress our file using lossless compession to reduce physical file size, without losing any information. In this case, we use `DEFLATE` with the default predictor of `2`.

We now have all the basic information required to make a COG that is suitablably performant for our leaflet app:

```bash
gdalwarp -of COG -co TILING_SCHEME=GoogleMapsCompatible -co COMPRESS=DEFLATE rgb.vrt rgb-cog.tif
```

If you didn't create a vrt you can replace `rgb.vrt` with your stacked tif already, e.g. `rgb.tif`.

## 3. Handling COG generation as part of a python microservice

Creating COGs on the command line is all well and good, however we wanted this conversion to be performed within our data processing pipeline, specifically after satellite data processing, or UAV orthomosaic generation and quality assurance tests are complete, therefore we need to handle the commands as part of a microservice that could be called whenever we have data we want to translate from a typical GeoTIFF to a COG, and store it. Here we have added some generalised extracts to show how that might look as part of a production microservice.

#### 1. Handle the command execution

We want to ensure that the command runs, and log any exceptions that might occur in a logbook for debugging later:

```python
from subprocess import run, CalledProcessError, CompletedProcess
from shutil import which


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

#### 2. Build the commands based on inputs and run

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

We are assuming that the file paths and additional paramters have been sent to the service, however we will use our simple example from above to put it all together:

```python
inputs_paths = ["red.tif", "green.tif", "blue.tif"]

output_cog_path = "rgb.tif"

run(input_paths, output_cog_path)

# Send `output_cog_path` to the cloud storage bucket.
```

If this completes successfully, we can send our resultant COG back to the cloud storage bucket where it can be requested by the application.


## 4. Hosting COGs for consumption
<!-- Section by SR -->

Very simple, stick it in a bucket that allows range requests. We use Google Cloud which is configured by default. There's no need for a web server if you have the ability to make range requests from the client.

Not something we do, but it's possible to generate map tiles or Mapbox vector tiles on the fly from your COGs. Take a look at [rio-tiler]() and [rio-mvt]() if you're interested.

## 5. Consuming COGs with React & leaflet
<!-- Section by SR -->

With the COG hosted ready read go, all we need is a way to display it. While the COG data is organised so making range requests is possible, the client app has the responsibility of making those requests. A client uses the maps view port and resulting zoom level to make these requests and then renders the result. The [GeoTIFF](https://github.com/GeoTIFF) project maintained by [Daniel Dufour](https://github.com/DanielJDufour) has libraries to do both fetching and rendering. The [georaster](https://github.com/GeoTIFF/georaster) library provides an interface for making range requests and the [georaster-layer-for-leaflet](https://github.com/GeoTIFF/georaster-layer-for-leaflet) plugin works along side it to render the COG on a Leaflet map.

We use React which means there are a few extra hoops for using Leaflet. Fortunately [react-leaflet](https://react-leaflet.js.org/) has excellent bindings and it's recent major release (v3) comes with a core api for building custom components - easily. The examples below make use of `react-leaflet` v3.x. If needed, [this Code Sandbox](https://codesandbox.io/s/react-leaflet-georaster-forked-tzl15) uses 2.x or see [this migration guide](https://sean-rennie.medium.com/migrating-react-leaflet-from-v2-to-v3-12d6088af191) - written by Sean.

### Writing a custom react-leaflet wrapper

The wrapper we created combines `georaster-layer-for-leaflet` and `georaster` into a single component. There are a few required dependencies to install for building the component.

```bash
yarn install georaster leaflet react-leaflet @react-leaflet/core georaster-layer-for-leaflet
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

`react-leaflets` core api provides high level factory functions for building Leaflet components. It abstracts away logic for adding and removing a layer from the map as well as adding and cleaning event listeners. There is a lot going on there but what's important is that `createPathComponent` allows the rendering of map elements by Leaflet while taking care of any issues React might cause - like unnecessarily adding and removing layers every render.

The only required option for `georaster-layer-for-leaflet` is `georasters` which takes an array of `Georaster` instances. It's this interface that's responsible making range requests to the COG file. When multiple `Georaster` instances are supplied it means we can render more than one COG, think band stacking but in the client. The `useGeoraster`  hook wraps the pieces required to kick off requests by feeding one `parseGeoraster` call per path into `Promise.all()`. If successful it will return an array of `Georaster` objects. We've also included some status indicators as well for completeness.

#### Putting GeoRasterLayer to use

Our `GeoRasterLayer` takes three important props, `paths`, `pixelValuesToColorFn` and `resolution`, together they fetch and control how the image is rendered. `paths` is an array of urls to accessing the COG(s).

The `resolution` determines how the sampling of pixels is done per tile. This [issue](https://github.com/GeoTIFF/georaster-layer-for-leaflet/issues/51) explains it better but whats worth noting is the higher the resolution the more intensive client CPU requirements are while render. We still figuring out the best value as it depends on our users. For now we are using `64` which looks like a good middle ground between performance and quality.

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

To render an image on screen we'll apply a colour to every pixel. The `pixelValuesToColorFn` gives us control over colour get applied to each pixel when the `canvas` is rendered. It's passed a `values` parameter which is an array of band values - it's dependant on the COG. If the image is RGB the values will be an array of three bands: `[ R, G, B ]`. If it's a single band `NIR` image then the value will be an array with one value. When multiple images are used (by using multiple paths) the values argument combines both. The value order depends on that of the urls supplied to `paths`. For example, if `paths={[ NIR_URL, RED_URL ]}` the resulting values argument would be `[ NIR, Red ]`. `pixelValuesToColorFn` must return a valid [CSS colour](https://developer.mozilla.org/en-US/docs/Web/CSS/color) string.

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

## 6. Performance
<!-- Section by SR -->

The story for COGs at Hummingbird is about cost benefit. In our case, cloud compute and storage cost versus higher resolution images in our web platform. User experience is also key.

Diving into storage and processing performance we noticed map tiles become more expensive than COGs as soon as you tile over zl 20. As we've mentioned we need resolution down to zl 24 or 25 for our 5cm GSD products. Some back of the envelope calculations show we'd be saving up to 45% in storage costs by using COGs. There's another win there too, when its all said and done, you only need one remaining file - the COG. No need to store the tile set and the original GeoTIFF.
<!-- Something about benefit of VRTs? -->
 When it comes to processing, COGs are often up to 10x faster to generate than tiles. In our books this is a massive win.

Measuring on platform performance is a bit trickier and subjective. What is a considered a "fast" load to a user navigating a lettuce field. Is that the most important metric or is seeming greater detail at lower zoom level important. We are actively evaluating this question, but for the time being we are happy with the perceived performance of COGs when compare to map tiles. If we get a more concrete answer we'll be sure to write up a new post on our learnings.
