---
title:  "How to dynamically generate SVGs with plain JavaScript and React"
comments: true
---
# Dynamically generate SVGs with plain JavaScript and React

Embedded SVGs on websites are a powerful design feature of modern web development.
With SVGs you can create sophisticated high-quality [interactive data visualizations](https://archive.nytimes.com/www.nytimes.com/interactive/2013/03/29/sports/baseball/Strikeouts-Are-Still-Soaring.html?ref=baseball).

But sometimes, all you want to do is put some user-provided text on a rectangle.
As an example, here is a cuboid of dirt with a volume in liters.

<div style="text-align: center">
  <img alt="Box with volume" src="/assets/2019-05-30/volume1.svg" width="250" />
</div>

I'll show you three methods how to do that without specialized libraries.

## Method 1: Using template literals and data-URI

1. Create an image in your favourite vector graphics editor (Inkscape) and save it as
   optimized SVG.
2. Copy the SVG contents within a JavaScript template literal `` `...` `` and replace the text you want to change dynamically.
   In the following example to: `${amount}&#8467;`
3. Convert the string into a data-uri with `"data:image/svg+xml;base64," + btoa(source)` and use it as image URL.

<script async src="//jsfiddle.net/gubser/7mjxv5b3/embed/result,js,html/"></script>

This method is useful for creating dynamic markers in Leaflet maps:
<script async src="//jsfiddle.net/gubser/hy4ow3xa/embed/result,js,html/"></script>



## Method 2: Manipulating SVG in DOM
Instead of rendering a template string to a data URI, you can add the SVG directly in your website and
manipulate it using the DOM. In this example, I've given the `tspan`-Element the ID `volume`.

<script async src="//jsfiddle.net/gubser/7qug43zb/embed/result,js,html/"></script>

## Method 3: Create SVG using React
React with Typescript has full support for SVG elements.
The following demo project renders a timeline visualization of a sequence of nodes.
The diagram consists of labels, minor guides, major guides and a line for each node.


<svg width="400px" height="265px" viewBox="0 0 400 265" preserveAspectRatio="xMinYMin meet"><line x1="46" y1="25" x2="46" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="62" y1="25" x2="62" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="77" y1="25" x2="77" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="92" y1="25" x2="92" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="108" y1="25" x2="108" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="123" y1="25" x2="123" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="138" y1="25" x2="138" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="154" y1="25" x2="154" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="169" y1="25" x2="169" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="185" y1="25" x2="185" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="200" y1="25" x2="200" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="215" y1="25" x2="215" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="231" y1="25" x2="231" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="246" y1="25" x2="246" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="262" y1="25" x2="262" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="277" y1="25" x2="277" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="292" y1="25" x2="292" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="308" y1="25" x2="308" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="323" y1="25" x2="323" y2="265" stroke="#CCC" stroke-width="1"></line><line x1="338" y1="25" x2="338" y2="265" stroke="#CCC" stroke-width="1"></line><text x="46" y="20" text-anchor="middle">0</text><line x1="46" y1="25" x2="46" y2="265" stroke="#666" stroke-width="1"></line><text x="108" y="20" text-anchor="middle">20</text><line x1="108" y1="25" x2="108" y2="265" stroke="#666" stroke-width="1"></line><text x="169" y="20" text-anchor="middle">40</text><line x1="169" y1="25" x2="169" y2="265" stroke="#666" stroke-width="1"></line><text x="231" y="20" text-anchor="middle">60</text><line x1="231" y1="25" x2="231" y2="265" stroke="#666" stroke-width="1"></line><text x="292" y="20" text-anchor="middle">80</text><line x1="292" y1="25" x2="292" y2="265" stroke="#666" stroke-width="1"></line><text x="354" y="20" text-anchor="middle">100</text><line x1="354" y1="25" x2="354" y2="265" stroke="#666" stroke-width="1"></line><g><polygon points="83,45 98,85 197,85 182,45" fill="#2c28"></polygon><line x1="62" x2="200" y1="45" y2="45" stroke="#222" stroke-width="2"></line><line x1="62" x2="62" y1="41" y2="49" stroke="#222" stroke-width="2"></line><line x1="200" x2="200" y1="41" y2="49" stroke="#222" stroke-width="2"></line><line x1="83" x2="98" y1="45" y2="85" stroke="#22d" stroke-width="2"></line><line x1="182" x2="197" y1="45" y2="85" stroke="#d22" stroke-width="2"></line></g><g><polygon points="98,85 129,125 200,125 197,85" fill="#2c28"></polygon><line x1="62" x2="200" y1="85" y2="85" stroke="#222" stroke-width="2"></line><line x1="62" x2="62" y1="81" y2="89" stroke="#222" stroke-width="2"></line><line x1="200" x2="200" y1="81" y2="89" stroke="#222" stroke-width="2"></line><line x1="98" x2="129" y1="85" y2="125" stroke="#22d" stroke-width="2"></line><line x1="197" x2="200" y1="85" y2="125" stroke="#d22" stroke-width="2"></line></g><g><polygon points="129,125 138,165 243,165 200,125" fill="#2c28"></polygon><line x1="108" x2="262" y1="125" y2="125" stroke="#222" stroke-width="2"></line><line x1="108" x2="108" y1="121" y2="129" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="121" y2="129" stroke="#222" stroke-width="2"></line><line x1="129" x2="138" y1="125" y2="165" stroke="#22d" stroke-width="2"></line><line x1="200" x2="243" y1="125" y2="165" stroke="#d22" stroke-width="2"></line></g><g><polygon points="138,165 154,205 255,205 243,165" fill="#2c28"></polygon><line x1="108" x2="262" y1="165" y2="165" stroke="#222" stroke-width="2"></line><line x1="108" x2="108" y1="161" y2="169" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="161" y2="169" stroke="#222" stroke-width="2"></line><line x1="138" x2="154" y1="165" y2="205" stroke="#22d" stroke-width="2"></line><line x1="243" x2="255" y1="165" y2="205" stroke="#d22" stroke-width="2"></line></g><g><polygon points="154,205 154,245 255,245 255,205" fill="#2c28"></polygon><line x1="138" x2="262" y1="205" y2="205" stroke="#222" stroke-width="2"></line><line x1="138" x2="138" y1="201" y2="209" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="201" y2="209" stroke="#222" stroke-width="2"></line><line x1="154" x2="154" y1="205" y2="245" stroke="#22d" stroke-width="2"></line><line x1="255" x2="255" y1="205" y2="245" stroke="#d22" stroke-width="2"></line></g><g><line x1="138" x2="262" y1="245" y2="245" stroke="#222" stroke-width="2"></line><line x1="138" x2="138" y1="241" y2="249" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="241" y2="249" stroke="#222" stroke-width="2"></line></g></svg>


The blue line connects all `start`, the red line connects all `end`.
Each node is visualized as a row and has the following properties:
 - `min`, `max` - The earliest and latest allowed time
 - `start`, `end` - The earliest and latest possible time


<iframe src="https://codesandbox.io/embed/delicate-star-ub4d8?fontsize=14" title="Create SVG using React" style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;" sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"></iframe>

Using React [function components](https://reactjs.org/docs/components-and-props.html#function-and-class-components),
it's very easy to organize the different geometry producing logic.
And if you use Typescript, your SVG-code is statically type checked. 👍👍

For example the label is created like this:
```tsx
function TimeLabel({ tx, mins }) {
  return (
    <text x={tx(mins)} y={timeLabelY} textAnchor="middle">
      {mins}
    </text>
  );
}
```

and the minor guide like this:
```tsx
function MinorTimeGuide({ tx, height, mins }) {
  return (
    <line
      x1={tx(mins)}
      y1={headerHeight}
      x2={tx(mins)}
      y2={height}
      stroke={colorMinorTimeGuide}
      strokeWidth="1"
    />
  );
}
```

### Stretching issues
<div style="text-align: center">
<svg width="200px" height="225px" viewBox="0 0 400 225" preserveAspectRatio="none"><line x1="46" y1="25" x2="46" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="62" y1="25" x2="62" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="77" y1="25" x2="77" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="92" y1="25" x2="92" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="108" y1="25" x2="108" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="123" y1="25" x2="123" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="138" y1="25" x2="138" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="154" y1="25" x2="154" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="169" y1="25" x2="169" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="185" y1="25" x2="185" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="200" y1="25" x2="200" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="215" y1="25" x2="215" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="231" y1="25" x2="231" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="246" y1="25" x2="246" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="262" y1="25" x2="262" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="277" y1="25" x2="277" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="292" y1="25" x2="292" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="308" y1="25" x2="308" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="323" y1="25" x2="323" y2="225" stroke="#CCC" stroke-width="1"></line><line x1="338" y1="25" x2="338" y2="225" stroke="#CCC" stroke-width="1"></line><text x="46" y="20" text-anchor="middle">0</text><line x1="46" y1="25" x2="46" y2="225" stroke="#666" stroke-width="1"></line><text x="108" y="20" text-anchor="middle">20</text><line x1="108" y1="25" x2="108" y2="225" stroke="#666" stroke-width="1"></line><text x="169" y="20" text-anchor="middle">40</text><line x1="169" y1="25" x2="169" y2="225" stroke="#666" stroke-width="1"></line><text x="231" y="20" text-anchor="middle">60</text><line x1="231" y1="25" x2="231" y2="225" stroke="#666" stroke-width="1"></line><text x="292" y="20" text-anchor="middle">80</text><line x1="292" y1="25" x2="292" y2="225" stroke="#666" stroke-width="1"></line><text x="354" y="20" text-anchor="middle">100</text><line x1="354" y1="25" x2="354" y2="225" stroke="#666" stroke-width="1"></line><g><polygon points="83,45 98,85 197,85 182,45" fill="#2c28"></polygon><line x1="62" x2="200" y1="45" y2="45" stroke="#222" stroke-width="2"></line><line x1="62" x2="62" y1="41" y2="49" stroke="#222" stroke-width="2"></line><line x1="200" x2="200" y1="41" y2="49" stroke="#222" stroke-width="2"></line><line x1="83" x2="98" y1="45" y2="85" stroke="#22d" stroke-width="2"></line><line x1="182" x2="197" y1="45" y2="85" stroke="#d22" stroke-width="2"></line></g><g><polygon points="98,85 129,125 200,125 197,85" fill="#2c28"></polygon><line x1="62" x2="200" y1="85" y2="85" stroke="#222" stroke-width="2"></line><line x1="62" x2="62" y1="81" y2="89" stroke="#222" stroke-width="2"></line><line x1="200" x2="200" y1="81" y2="89" stroke="#222" stroke-width="2"></line><line x1="98" x2="129" y1="85" y2="125" stroke="#22d" stroke-width="2"></line><line x1="197" x2="200" y1="85" y2="125" stroke="#d22" stroke-width="2"></line></g><g><polygon points="129,125 138,165 243,165 200,125" fill="#2c28"></polygon><line x1="108" x2="262" y1="125" y2="125" stroke="#222" stroke-width="2"></line><line x1="108" x2="108" y1="121" y2="129" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="121" y2="129" stroke="#222" stroke-width="2"></line><line x1="129" x2="138" y1="125" y2="165" stroke="#22d" stroke-width="2"></line><line x1="200" x2="243" y1="125" y2="165" stroke="#d22" stroke-width="2"></line></g><g><polygon points="138,165 154,205 255,205 243,165" fill="#2c28"></polygon><line x1="108" x2="262" y1="165" y2="165" stroke="#222" stroke-width="2"></line><line x1="108" x2="108" y1="161" y2="169" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="161" y2="169" stroke="#222" stroke-width="2"></line><line x1="138" x2="154" y1="165" y2="205" stroke="#22d" stroke-width="2"></line><line x1="243" x2="255" y1="165" y2="205" stroke="#d22" stroke-width="2"></line></g><g><line x1="138" x2="262" y1="205" y2="205" stroke="#222" stroke-width="2"></line><line x1="138" x2="138" y1="201" y2="209" stroke="#222" stroke-width="2"></line><line x1="262" x2="262" y1="201" y2="209" stroke="#222" stroke-width="2"></line></g></svg>
</div>

At first, I wanted to use some arbitrary coordinate system and then stretch the SVG to fit in a website
using `preserveAspectRatio="none"` (See [SVG Pocket Guide](http://svgpocketguide.com/book/#section-1)).
But then I realized that all the text is stretched too. To avoid this, each component needs to calculate the fitted x-coordinates
manually. To do that, I'm giving a small function `tx(value: number): number` as a prop to all components

## Links
- [Pocket Guide to Writing SVG](http://svgpocketguide.com/book/) - Superb guide about SVG elements and file format.
- [D3](https://d3js.org/) - DOM manipulation with a focus on interactive data visualization, huge ecosystem.
- [Lottie](http://airbnb.io/lottie/) - Render stunning animations, looks great.


