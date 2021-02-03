# Migrating react-leaflet from v2 to v3

*[Paul LeCam's](https://github.com/PaulLeCam) [react-leaflet](https://react-leaflet.js.org/) library is a powerful set of react bindings for [Leaflet](https://leafletjs.com/index.html). Recently the library has been overhauled to modern react, making use of hooks and doing away completely with classes. It's also been re-written in typescript 🎉! In my view this work makes `react-leaflet` more of a direct binding to `leaflet` rather than a "react flavoured" of it. The introduction of a core library makes building custom components really easy. It gives react developers the full power of leaflet and its [plugin ecosystem](https://leafletjs.com/plugins.html).*

*Hats off to you Paul, thanks for the amazing work. Below is my attempt to add something to it by outlining the changes our app needed in order to migrate to v3.*

## Some Initial Assumptions

This guide is written for those already comfortable with `leaflet` and `react-leaflet`. The examples are written in Typescript where applicable but shouldn't be a blocker for Javascript migrations. Your application is written with or migrating to react hooks and functional components.

## So whats changed in V3.x?
