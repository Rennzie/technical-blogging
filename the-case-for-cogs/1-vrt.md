# Creating a GDAL Virtual Format (VRT) in preparation to create a stacked COG

- Written By: AJ Hitchins

## Aim

Generate a VRT from multiple images in order to create a stacked COG

- Plan:
  - Multiple image bands describing a scene at a given wavelength
  - Ensure band order is correct
  - Build VRT from bands
  - VRT is prepared for COG generation

## VRT Overview

The [Geospatial Data Abstraction Library (GDAL)](https://gdal.org/) is a well established translator library for
geospatial data formats. One of the features of GDAL enables raster datasets to be created virutally from other raster datasets. The VRT file is an XML format that references the raster layers and associated metadata to
describe where they are positioned geographically.

In this example we are using GDAL version `3.2.1`

## Why are we using VRTs to generate COGs?

In our use case, we followed the same logic that many public earth observation datasets have followed, which keeps each image band of a given wavelength separate. In the case of Sentinel-2 data, you can see the configurations [here](https://sentinel.esa.int/web/sentinel/technical-guides/sentinel-2-msi/msi-instrument).

In order to stack the bands into a multi-band image file, we will generate a VRT to order the bands correcty, and generate a single COG from the VRT, referencing the three separate images.

If you prefer to keep the bands separate, then this step is not necessary to generate COGs.


## Writing a VRT

Writing a VRT is pretty straight forward - you can call the gdal command [`gdalbuildvrt`](https://gdal.org/programs/gdalbuildvrt.html?highlight=gdalbuildvrt). If you follow the link you can see an example of a VRT file
- they are typically no less than 50 lines long.
Given three separate datasets: `red.tif`, `green.tif`, `blue.tif` for example, we can build a simple VRT to create a separate band RGB image, `rgb.vrt`.
We want to keep the image bands separate to ensure that they don't get merged into a single band in the dataset.

```bash
gdalbuildvrt -separate rgb.vrt red.tif green.tif blue.tif
```

This creates a simple VRT for a RGB dataset. The example below shows the contents of a VRT for a UAV orthomosaic in
UTM Zone 14N consisting of the three separate datasets: `red.tif`, `green.tif` and `blue.tif`.
