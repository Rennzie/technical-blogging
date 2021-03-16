# Intro/TLDR

- Written by: Sean & Al,

- Plan:
  - Like an exec summary, should have all the take aways of reading the whole blog
  - What it is, why use it,
  - Our use cause
    - High resolution imagery at a lower cost
  - COG vs Raster tiles

Cloud Optimised GeoTIFFs (COGs) leverage HTTP GET range requests making them a lower cost alternative to traditional map tiles. In our view they are equally as performant, with the advantage of being less power hungry during processing (upto 10x) and require less storage (savings of 40%-80%) for the equivalent resolution. They are also more versatile. COGs are simply GeoTIFFs and can be used for image processing in all the usual ways - you don't need the "original" TIFF too. Map tiles, on the other hand, are more specific to web applications. If you need to process that image again you'll have to store tiles and TIFFs.

The use of COG's at Hummingbird was born from the need to display high resolution drone imagery on our platform. We've been using map tiles, but to save on processing we'd only tiled them to zoom level 20. New products use imagery up to 5cm GSD for counting lettuces or picking out weeds in an empty field. These images resolve down to zoom level 25 so being the cost sensitive start-up we are COGs became an obvious choice.

Our go to is to generate the COGs with `gdal`. We first make Virtual Raster Tiles (VRTS) from multiple source GeoTIFFs. This replaces the need to stacking as we don't need the stacked TIFF. A COG is generated from the VRT and "hosted" in a Google Cloud bucket. Our React application is the primary user of the COGs. We leverage `georaster-layer-for-leaflet` and `georaster` javascript libraries to build a custom `react-leaflet` component for displaying the COGs. 


