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
geospatial data formats. One of the features of GDAL enables raster datasets to be created virutally from other raster
datasets. The VRT file is an XML format that references the raster layers and associated metadata to describe where they
are geosptially positioned.

<GDAL version?>

## Why are using VRTs to generate COGs?

In our use case, we followed the same logic that many public earth observation datasets have followed, which 

## Writing a VRT

Writing a VRT is pretty straight forward - you can call the gdal command [`gdalbuildvrt`](https://gdal.org/programs/gdalbuildvrt.html?highlight=gdalbuildvrt). Given three separate datasets: `red.tif`, `green.tif`, `blue.tif` for example, we can
build a simple VRT to create a separate band RGB image, `rgb.vrt`.
We want to keep the image bands separate to ensure that they don't get merged into a single band in the dataset.

```bash
gdalbuildvrt -separate rgb.vrt red.tif green.tif blue.tif
```

This creates a simple VRT for a RGB dataset. The example below shows the contents of a VRT for a UAV orthomosaic in
UTM Zone 14N consisting of the three separate datasets: `red.tif`, `green.tif` and `blue.tif`.

Some location data has been changed.

```
<VRTDataset rasterXSize="20020" rasterYSize="19180">
  <SRS>PROJCS["WGS 84 / UTM zone 14N",GEOGCS["WGS 84",DATUM["WGS_1984",SPHEROID["WGS 84",6378137,298.257223563,AUTHORITY["EPSG","7030"]],AUTHORITY["EPSG","6326"]],PRIMEM["Greenwich",0,AUTHORITY["EPSG","8901"]],UNIT["degree",0.0174532925199433,AUTHORITY["EPSG","9122"]],AUTHORITY["EPSG","4326"]],PROJECTION["Transverse_Mercator"],PARAMETER["latitude_of_origin",0],PARAMETER["central_meridian",-99],PARAMETER["scale_factor",0.9996],PARAMETER["false_easting",500000],PARAMETER["false_northing",0],UNIT["metre",1,AUTHORITY["EPSG","9001"]],AXIS["Easting",EAST],AXIS["Northing",NORTH],AUTHORITY["EPSG","32614"]]</SRS>
  <GeoTransform>  1.1234000000000000e+05,  1.0240000000000001e-02,  0.0000000000000000e+00,  1.1234000000000000e+06,  0.0000000000000000e+00, -1.0240000000000001e-02</GeoTransform>
  <VRTRasterBand dataType="Byte" band="1">
    <NoDataValue>0</NoDataValue>
    <ComplexSource>
      <SourceFilename relativeToVRT="1">red.tif</SourceFilename>
      <SourceBand>1</SourceBand>
      <SourceProperties RasterXSize="20020" RasterYSize="19180" DataType="Byte" BlockXSize="256" BlockYSize="256" />
      <SrcRect xOff="0" yOff="0" xSize="20020" ySize="19180" />
      <DstRect xOff="0" yOff="0" xSize="20020" ySize="19180" />
      <NODATA>0</NODATA>
    </ComplexSource>
  </VRTRasterBand>
  <VRTRasterBand dataType="Byte" band="2">
    <NoDataValue>0</NoDataValue>
    <ComplexSource>
      <SourceFilename relativeToVRT="1">green.tif</SourceFilename>
      <SourceBand>1</SourceBand>
      <SourceProperties RasterXSize="20020" RasterYSize="19180" DataType="Byte" BlockXSize="256" BlockYSize="256" />
      <SrcRect xOff="0" yOff="0" xSize="20020" ySize="19180" />
      <DstRect xOff="0" yOff="0" xSize="20020" ySize="19180" />
      <NODATA>0</NODATA>
    </ComplexSource>
  </VRTRasterBand>
  <VRTRasterBand dataType="Byte" band="3">
    <NoDataValue>0</NoDataValue>
    <ComplexSource>
      <SourceFilename relativeToVRT="1">blue.tif</SourceFilename>
      <SourceBand>1</SourceBand>
      <SourceProperties RasterXSize="20020" RasterYSize="19180" DataType="Byte" BlockXSize="256" BlockYSize="256" />
      <SrcRect xOff="0" yOff="0" xSize="20020" ySize="19180" />
      <DstRect xOff="0" yOff="0" xSize="20020" ySize="19180" />
      <NODATA>0</NODATA>
    </ComplexSource>
  </VRTRasterBand>
</VRTDataset>
```
