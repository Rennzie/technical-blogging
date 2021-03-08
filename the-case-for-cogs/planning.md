# Writing Plan

## Title

- The case fo COGs in web applications

## Blurb

How to leverage Cloud Optimised GeoTIFFs for displaying high resolution drone imagery on a web application while saving storage costs.

## Authors

- Alasdair Hitchins
- Sean Rennie

## Ideas

- Implementing COG's in a production environment
- Generating COG's with GDAL
- Consumming a COG in a React application
- COG vs Map Tiles
- Low level explanation of what COGs are and what their benefits are

## Intro (SR)

## TOC

### What is a Cloud Optimsed GeoTIFF (COG) - (AH)

...

### Benefits of COGs - What makes them different and who should use them? (SR & AH)

...

### 1. Generating VRTs from separate rasters for COG creation

...

### 2. Creating COGs using GDAL

...

<!-- Not sure we need section 3 unless there is something specific we need to configure in GCP? "Benefitting from their structure" would likely be covered in "Benefits of COGs" above! -->
### 3. Hosting COGs on the cloud & benefitting from their structure

...

### 4. Consuming COGs in a React application (SR)

...

### 5. Performance/ Why choose COGs (SR)

- How do we measure performance of COG vs raster tiles?
- Argument can be made for less storage space using COG's and less redundancy because you only need the cog, not the tiff and tiles.
...

<!-- Might be worth including a Glossary for all the acronyms -->
### Glossary

## Open Questions

- Do we write this as a multi part blog?
- Should we focus on implementation or explanation of what COG's are or both (multi part)?
