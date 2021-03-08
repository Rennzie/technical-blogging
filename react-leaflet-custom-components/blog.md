# Using Leaflet VectorGrid to render vector tiles in a React app

*A look at building custom react-leaflet components for Leaflet plugins*

Leaflet is a powerful open source mapping library, but (by design) it has its limits. It's also designed to be extensible and there is a vast ecosytem of plugins available for almost every use case. This is great news unless you're a React developer. Both React and Leaflet want to control the DOM, both by different mechanisms. Fortunately the [react-leaflet](https://react-leaflet.js.org) library provides fantastic bindings to Leaflet, making it really easy to use in our React apps.

`react-leaflets`'s recent major version change is a complete re-write of the library. It leverages hooks (an awesome example of their power), is written in Typescript, and most importantly has a new core library [@react-leaflet/core](). If you still need to migrate, take a look at my previous blog covering [the what and the how]().

The core library allows developers to easily build Leaflet components. There are low level hooks for adding and removing layers from the map along with adding and cleaning event listeners. High level factories tie the hooks together and in the simplest case make writing a custom component less than TK 10 lines.

The documentation gives a thorough [overview](https://react-leaflet.js.org/docs/core-architecture) on how all these pieces fit together. My aim is not to repeat that work but to give a different perspective on it. I'll be walking through how I used the core library to build a custom [Leaflet VectorGrid](https://github.com/Leaflet/Leaflet.VectorGrid) component for rendering vector tiles. I also touch on extending the plugin for a particular use case we needed.  

## Background on the problem
