---
title:  "Dynamically render SVGs"
comments: true
---
# Dynamically create SVGs using plain JavaScript and React
Embedded SVGs on websites are a powerful feature of modern web development.
Not only are they perfectly sharp no matter the resolution, but more importantly,
they can be easily created or modified in the browser to highly informative dynamic content.
I will show you three methods to do that without specialized libraries.


## Method 1: Using template literals and data-URI
<img alt="Box with volume" src="/assets/2019-05-30/volume1.svg" width="250" />

1. Create an image in your favourite vector graphics editor (Inkscape) and save it as
   optimized SVG.
2. Copy the SVG contents within a JavaScript template literal `` `...` `` and replace the text you want to change dynamically.
   In the following example to: `${amount}&#8467;`
3. Convert the string into a data-uri with `"data:image/svg+xml;base64," + btoa(source)` and use it as image URL.

<script async src="//jsfiddle.net/gubser/7mjxv5b3/embed/result,js,html/"></script>

This method is useful for creating dynamic markers in Leaflet maps:
<script async src="//jsfiddle.net/gubser/hy4ow3xa/embed/result,js,html,css/"></script>



## Method 2: Manipulating SVG in DOM
Instead of rendering a template string to a data URI, you can add the SVG directly in your website and
manipulate it using the DOM.

<script async src="//jsfiddle.net/gubser/7qug43zb/embed/result,js,html/"></script>

## Method 3: Create SVG using React
React with Typescript has full support for SVG elements.
The following example renders a timeline visualization for visiting a set of nodes.
Each node has the following properties:
 - min, max - The earliest and latest allowed time
 - start, end - The earliest and latest possible time

 The visualization draws a blue line connecting the earliest possible times when visiting the nodes.
 The red line connect the latest possible times.

<iframe src="https://codesandbox.io/embed/delicate-star-ub4d8?fontsize=14" title="delicate-star-ub4d8" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

## Links
- [Pocket Guide to Writing SVG](http://svgpocketguide.com/book/) - Superb guide about SVG elements and file format 👍
- [D3](https://d3js.org/) - Interactive data visualization, huge ecosystem
- [Lottie](http://airbnb.io/lottie/) - Render stunning animations.


