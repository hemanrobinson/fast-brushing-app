# Optimized Brushing

```html
<style>
h2 {
  padding-top: 10px;
}
rect.selection {
  fill: #bcd6fB;
  stroke: #bcd6fB;
  fill-opacity: 0.3;
  stroke-opacity: 1;
}
</style>

<p>
  Brushing is a powerful technique of interactive graphics, highlighting linked data in multiple views. When it was developed in the 1980s, brushing was limited to a few hundred points.  On modern hardware, we can brush many more.
</p>
<p>
  Drag the brush to select the points. Drag the edges to resize the brush. Drag a rectangle in any plot to create a new brush.
</p>
<p>
  Extend and reduce your selection by pressing the Shift and Control keys. (On a Mac, use the Shift and Command keys.)
</p>
<p>
  Adjust the sliders to change the number of points and their transparency.
</p>
```

```js
// Initialization.
const width = 200,
  height = 200,
  nColumns = Data.getColumnNames().length,
  powerNData = getPower( nData ),
  data = Data.getValues( powerNData ),
  totalWidth = nColumns * width,
  totalHeight = nColumns * height,
  brushNodeOffset = 4;
        
/**
* Returns a function, that, as long as it continues to be invoked, will not be triggered.
* The function will be called after it stops being called for `wait` milliseconds.
*
* From https://levelup.gitconnected.com/debounce-in-javascript-improve-your-applications-performance-5b01855e086.
*
* @param func  function
* @param wait  delay, in milliseconds
* @return debounced function
*/
const debounce = ( func, wait ) => {
  let timeout;
  return function executedFunction( ...args ) {
    const later = () => {
      clearTimeout( timeout );
      func( ...args );
    };
    clearTimeout( timeout );
    timeout = setTimeout( later, wait );
  };
};

// Cache scaled coordinates.
Matrix.scaled = [];
let scale = [];
for( let i = 0; ( i < nColumns ); i++ ) {
  Matrix.scaled[ i ] = new Uint16Array( powerNData );
  let x = i * width;
  scale[ i ] = d3.scaleLinear().domain( Data.getDomain( powerNData, i )).range([ x + Plot.padding, x + width - Plot.padding ]);
}
data.forEach(( datum, row ) => {
  for( let i = 0; ( i < nColumns ); i++ ) {
    let x = i * width;
    Matrix.scaled[ i ][ row ] = Math.round( scale[ i ]( datum[ i ]) - x );
  }
});

// Create the DIV and CANVAS elements.
const div = d3.create( "div" );
const canvas = div.append( "canvas" )
  .attr( "width", totalWidth )
  .attr( "height", totalHeight )
  .style( "position", "absolute" )
  .style( "border", "1px solid #939ba1" )
  .style( "background-color", "#ffffff" );
Matrix.canvas = canvas.node();

// Create the SVG elements (after https://observablehq.com/@d3/brushable-scatterplot-matrix?collection=@d3/d3-brush).
const svg = div.append( "svg" )
  .attr( "width", totalWidth )
  .attr( "height", totalHeight )
  .attr( "viewBox", [ 0, 0, totalWidth, totalHeight ])
  .style( "position", "relative" );
svg.selectAll( "*" ).remove();
const cell = svg.append( "g" )
  .selectAll( "g" )
  .data( d3.cross( d3.range( nColumns ), d3.range( nColumns )))
  .join( "g" )
  .attr( "transform", ([ i, j ]) => `translate(${ i * width },${ j * height })` );

// Create the brush.
const onStart = ( event ) => {
  if( event.sourceEvent ) {
    Matrix.isExtending = event.sourceEvent.shiftKey && !event.sourceEvent.ctrlKey && !event.sourceEvent.metaKey;
    Matrix.isReducing = !event.sourceEvent.shiftKey && ( event.sourceEvent.ctrlKey || event.sourceEvent.metaKey );
    const target = event.sourceEvent.target.parentNode;
    if( Matrix.brushNode !== target ) {
      d3.select( Matrix.brushNode ).call( brush.move, null );
      Matrix.brushNode = target;
      if( !Matrix.isExtending && !Matrix.isReducing ) {
        Data.deselectAll();
      }
    } else if( event.selection ) {
      const xDown = event.selection[ 0 ][ 0 ],
      yDown = event.selection[ 0 ][ 1 ],
      xUp = event.selection[ 1 ][ 0 ],
      yUp = event.selection[ 1 ][ 1 ];
      let offsetX, offsetY;
      if( event.sourceEvent.touches ) {
        const touch = event.sourceEvent.touches[ 0 ];
        offsetX = touch.clientX - Matrix.canvas.getBoundingClientRect().x;     
        offsetY = touch.clientY - Matrix.canvas.getBoundingClientRect().y;
      } else {
        offsetX = event.sourceEvent.offsetX;
        offsetY = event.sourceEvent.offsetY;
      }
      let i = Math.floor( offsetX / width ),
      j = Math.floor( offsetY / height ),
      x = i * width,
      y = j * height;
      if( !Matrix.isExtending && !Matrix.isReducing && !Plot.isWithin({ x: offsetX, y : offsetY }, { x: x + xDown, y: y + yDown, width: xUp - xDown, height: yUp - yDown })) {
        Data.deselectAll();
      }
    }
  }
};
const debouncedDraw = debounce( Matrix.draw, 1 );
const onBrush = ( event ) => {
  if( event.selection ) {
    const xDown = event.selection[ 0 ][ 0 ],
    yDown = event.selection[ 0 ][ 1 ],
    xUp = event.selection[ 1 ][ 0 ],
    yUp = event.selection[ 1 ][ 1 ],
    nColumns = Data.getColumnNames().length;
    let offsetX, offsetY;
    if( event.sourceEvent ) {
      Matrix.isExtending = event.sourceEvent.shiftKey && !event.sourceEvent.ctrlKey && !event.sourceEvent.metaKey;
      Matrix.isReducing = !event.sourceEvent.shiftKey && ( event.sourceEvent.ctrlKey || event.sourceEvent.metaKey );
      if( event.sourceEvent.touches ) {
        const touch = event.sourceEvent.touches[ 0 ];
        offsetX = touch.clientX - Matrix.canvas.getBoundingClientRect().x;     
        offsetY = touch.clientY - Matrix.canvas.getBoundingClientRect().y;
      } else {
        offsetX = event.sourceEvent.offsetX;
        offsetY = event.sourceEvent.offsetY;
      }
    } else {      
      offsetX = width * Math.floor( brushNodeOffset / 4 );
      offsetY = height * ( brushNodeOffset % 4 );
    }
    let i = Math.floor( offsetX / width ),
      j = Math.floor( offsetY / height ),
      x = i * width,
      y = j * height;
    if(( i !== j ) && ( 0 <= i ) && ( i < nColumns ) && ( 0 <= j ) && ( j < nColumns )) {
      const selectedRows = Plot.select( x, y, width, height, i, j, Matrix.scaled, { x: x + xDown, y: y + yDown, width: xUp - xDown, height: yUp - yDown });
      if( Matrix.isExtending || Matrix.isReducing ) {
        const set = new Set( selectedRows );    // faster than Arrays
        const reducedRows = Data.selectedRows.filter(( item ) => !set.has( item ));
        Data.selectedRows = Matrix.isReducing ? reducedRows : selectedRows.concat( reducedRows );
      } else {
        Data.selectedRows = selectedRows;
      }
      if( Matrix.bitmaps && Matrix.bitmaps[ i ]) {
        Plot.draw( x, y, width, height, i, j, Matrix.scaled, Matrix.canvas, 1 - transparency, Data.selectedRows, Matrix.bitmaps[ i ][ j ]);
      }
    } 
  }
  debouncedDraw( width, height, powerNData, 1 - transparency, false );
};
const onEnd = ( event ) => {
  Matrix.draw( width, height, powerNData, 1 - transparency, true );
};
const brush = d3.brush()
  .extent([[ 2, 2 ], [ width, height ]])
  .keyModifiers( false )
  .on( "start", onStart )
  .on( "brush", onBrush )
  .on( "end", onEnd );
cell.call( brush );

// Initialize the brush.
Matrix.brushNode = svg.node().firstChild.childNodes[ brushNodeOffset ];
const brushCell = d3.select( Matrix.brushNode );
brushCell.call( brush.move, [[ 40, 40 ], [ 80, 80 ]]);

// Display the graph.
display( div.node());
```

```js
// Create the sliders.
const nData = view( Inputs.range([9, 18], {value: 14, step: 1, label: "Points per Plot:", format: ( value ) => getPower( value )}));
const transparency = view( Inputs.range([0, 0.99], {value: 0.5, step: 0.01, label: "Transparency:" }));
```

```js
// Listening to the sliders should not be necessary, but apparently it is.
const inputs = document.getElementsByTagName( "input" );
for( let i = 0; ( i < inputs.length ); i++ ) {
  inputs[ i ].addEventListener( "input", ( event ) => { Matrix.clear(); debouncedDraw; });
}
```

```html
<h2>User Interface</h2>

<p>
  This design is based on the <a href="http://www.sci.utah.edu/~kpotter/Library/Papers/becker:1987:BS/index.html">scatter plot matrix</a> of <a href="https://www.researchgate.net/scientific-contributions/Richard-A-Becker-7076158">Richard Becker</a> and <a href="https://www.cerias.purdue.edu/site/people/faculty/view/709">William Cleveland</a> (Becker and Cleveland, 1987).
</p>

<figure>
  <a href="https://www.datavis.ca/milestones/index.php?group=1975%2B&mid=ms259">
    <img title="Dr. Richard Becker" alt="Dr. Richard Becker" src="${await FileAttachment("becker.png").url()}" style="border: 1px solid">
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://www.datavis.ca/milestones/index.php?group=1975%2B&mid=ms259">
    <img title="Dr. William Cleveland" alt="Dr. William Cleveland" src="${await FileAttachment("cleveland.png").url()}" style="border: 1px solid">
  </a>
</figure>

<p>
  The goal of the scatter plot matrix is not to locate points, but to find patterns in the data.  Therefore, there are no axes, only data ranges.  This increases Tufte's "Data-Ink Ratio" (Tufte, 1983).
</p>
<p>
  Colors emphasize the data. Black on white gives maximum emphasis.  The red selection color draws attention. The grid, being less important, is gray.
</p>
<p>
  By the same logic, the brush could be gray. However, usability tests pointed out that the standard color for selection is blue (Ho, 2016).  Following standards eases the user's learning curve.
</p>
<p>
  The edge of the brush has a distinct color, indicating a distinct function: dragging the edge resizes the brush, while dragging within the brush moves it.  These distinct functions are reinforced by distinct cursor shapes, but cursor shapes are subtle and only appear on hover. The distinct colors are always clearly visible.
</p>
<p>
  <a href="https://github.com/d3/d3-brush">D3's brush</a> is <em>persistent</em> rather than <em>transient</em>.  A persistent brush reduces errors, by enabling the user to resize the brush (Tidwell, 2010).  A persistent brush also helps users share their explorations, through screen shots for example.
</p>
<p>
  Transparency shows density, via <a href="https://en.wikipedia.org/wiki/Alpha_compositing">alpha blending</a>.  This gives scatter plots the expressive power of contour plots, while still displaying individual points (Wegman and Luo, 2002).
</p>
<p>
  Shift, Control, and Command keys are standard modifiers to extend and reduce selections of individual objects (e.g. Apple, 2024). I have found no documented standards for their behavior during brushing. The behavior implemented here enables people to select irregular areas or disjoint clusters of points.
</p>

<h2>Implementation</h2>

<p>
  This project uses <a href="https://react.dev">React</a>, <a href="https://github.com/mui-org/material-ui">Material-UI</a>, and <a href="https://github.com/d3/d3">d3</a>, and reuses some code from the <a href="https://observablehq.com/collection/@d3/d3-brush">d3-brush collection</a>.
</p>
<p>
  Optimization was a joint effort with <a href="https://observablehq.com/@fil">Fil</a>, whose suggestions made this code much faster. There are a number of smaller optimizations, but these had the greatest effect:
</p>
<ol>
  <li>Drawing in a single CANVAS element is faster than drawing thousands of SVG elements.</li>
  <li>Drawing each data point as a single pixel displays large data sets with minimal drawing code.</li>
  <li>Caching pixel coordinates in integer Arrays eliminates scaling calculations during drawing and selection.</li>
  <li>Caching deselected points in bitmaps reduces drawing to a fast <a href="https://en.wikipedia.org/wiki/Bit_blit">bit blit</a>, followed by drawing the selected points.</li>
  <li><a href="https://levelup.gitconnected.com/debounce-in-javascript-improve-your-applications-performance-5b01855e086">Debouncing</a> the brushing interaction reduces drawing in large data sets.</li>
</ol>
<p>
  Graph-specific optimizations are justified when brushing is frequent, as it is in scatter plot matrices.  Caching and debouncing can improve performance in any type of graph.
</p>
<p>
  Performance varies on different devices. My iMac (2020, 3.6 GHz 10-Core Intel Core i9, 128 GB) can brush 1,000,000 points per plot. In a 4x4 matrix, that's twelve million points.  As our hardware improves, we'll see these numbers grow.
</p>

<h2>Further Reading</h2>
<ul>
  <li>Apple (2024). MacOS User Guide. <a href="https://support.apple.com/en-ae/guide/mac-help/mchlp1378/mac">https://support.apple.com/en-ae/guide/mac-help/mchlp1378/mac</a>.</li><br/>
  <li>Becker, R. and Cleveland, W. (1987). "Brushing Scatterplots". Technometrics. 29 (2): 127-142. <a href="https://doi.org/10.2307/1269768">https://doi.org/10.2307/1269768</a>.</li><br/>
  <li>Ho, Y. (2016). Personal communication. <a href="https://www.linkedin.com/in/yang-ho-94b14860/">https://www.linkedin.com/in/yang-ho-94b14860/</a></li><br/>
  <li>Tidwell, J. (2010). Designing Interfaces: Patterns for Effective Interaction Design, Second Edition, 312-314.  Sebastopol CA: O'Reilly Media. <a href="https://www.oreilly.com/library/view/designing-interfaces-3rd/9781492051954/">https://www.oreilly.com/library/view/designing-interfaces-3rd/9781492051954/</a>.</li><br/>
  <li>Tufte, E. (1983). The Visual Display of Quantitative Information, First Edition, 91-105.  Cheshire CN: Graphics Press. <a href="https://www.edwardtufte.com/tufte/">https://www.edwardtufte.com/tufte/</a>.</li><br/>
  <li>Wegman, E. and Luo, Q. (2002). "On Methods of Computer Graphics for Visualizing Densities". Journal of Computational and Graphical Statistics 11, (1), 137-162. <a href="https://doi.org/10.1198/106186002317375659">https://doi.org/10.1198/106186002317375659</a>.</li><br/>
</ul>
```

```js
/**
 * Data.
 *
 * @return component
 */
const Data = new Object();

/**
 * Array of indices of selected rows.
 *
 * @type {number[]}
 */
Data.selectedRows = [];

/**
 * Deselects all rows.
 */
Data.deselectAll = () => {
  Data.selectedRows = [];
};

/**
 * Returns column names.
 *
 * @return {string[]}  column names
 */
Data.getColumnNames = () => {
  return [ "A", "B", "A * B", "sin( A / B )" ];
};

/**
 * Returns domain of specified column.
 *
 * @param  {number}    nData  number of data values
 * @param  {number}    index  column index
 * @return {number[]}  domain of specified column
 */
Data.getDomain = ( nData, index ) => {
  return [ d3.min( Data.getValues( nData ), d => d[ index ]), d3.max( Data.getValues( nData ), d => d[ index ])];
};

/**
 * Data values.
 *
 * @type {number[]}
 */
Data.values = [];

/**
 * Returns data values.
 *
 * @param  {number}  nData  number of data values
 * @return {Array[]}  data values by row
 */
Data.getValues = ( nData ) => {
  if( Data.values.length !== nData ) {
    let f = d3.randomNormal( 0, 0.5 );
    Data.values = [];
    for( let i = 0; ( i < nData ); i++ ) {
      let a = f(), b = f();
      Data.values[ i ] = [ a, b, a * b, ( b === 0 ) ? 0 : Math.sin( a / b )];
    }
  }
  return Data.values;
};

/**
 * Returns "nice" power of ten:  rounded to 1, 2, 5, 10, 20, 50, etc.
 *
 * @param  {number}  exp  exponent
 * @return {number}  "nice" power of ten:  rounded to 1, 2, 5, 10, 20, 50, etc.
 */
const getPower = ( exp ) => {
  let m = (( exp % 3 ) === 0 ) ? 1 : (( exp % 3 ) === 1 ) ? 2 : 5;
  return m * ( 10 ** Math.floor( exp / 3 ));
}

// Matrix.
const Matrix = new Object();

/**
 * Bitmaps of deselected rows, cached for optimization, or undefined if none.
 *
 * @type {ImageData[][]|undefined}
 */
Matrix.bitmaps = undefined;

/**
 * CANVAS element, or undefined if none.
 *
 * @type {Element|undefined}
 */
Matrix.canvas = undefined;

/**
 * Node containing a brush, or undefined if none.
 *
 * @type {Node|undefined}
 */
Matrix.brushNode = undefined;

/**
 * Scaled coordinates, or undefined if none.
 *
 * @type {Uint16Array[]|undefined}}
 */
Matrix.scaled = undefined;

/**
 * True iff extending selection.
 *
 * @type {boolean}
 */
Matrix.isExtending = false;

/**
 * True iff reducing selection.
 *
 * @type {boolean}
 */
Matrix.isReducing = false;

/**
 * Clears data structures.
 */
Matrix.clear = () => {
  if( !Matrix.isExtending && !Matrix.isReducing ) {
    Data.deselectAll();
  }
  Matrix.bitmaps = undefined;
};

/**
 * Draws the grid, the plots, and the axes.
 *
 * @param  {number}  width          width in pixels
 * @param  {number}  height         height in pixels
 * @param  {number}  nData          number of data values
 * @param  {number}  opacity        alpha
 * @param  {boolean} isDrawingAll   true iff clearing and redrawing grid and axes
 */
Matrix.draw = ( width, height, nData, opacity, isDrawingAll ) => {

  // Initialization.  If no context, do nothing.
  let canvas = Matrix.canvas,
    g = canvas.getContext( "2d" ),
    nColumns = Data.getColumnNames().length;
  if( !g ) {
    return;
  }

  // If requested, clear the drawing area and draw the grid.
  if( isDrawingAll ) {
    g.clearRect( 0, 0, nColumns * width, nColumns * height );
    g.strokeStyle = "#939ba1";
    for( let i = 1; ( i < nColumns ); i++ ) {
      g.moveTo( i * width + 0.5, 0 );
      g.lineTo( i * width + 0.5, nColumns * height );
      g.moveTo( 0, i * height + 0.5 );
      g.lineTo( nColumns * width, i * height + 0.5 );
    }
    g.stroke();
  }
  
  // Draw the plots and the axes.  On first draw, store the bitmaps.
  let isFirstDraw = !Matrix.bitmaps;
  if( isFirstDraw ) {
    Matrix.bitmaps = [];
  }
  for( let i = 0; ( i < nColumns ); i++ ) {
    for( let j = 0; ( j < nColumns ); j++ ) {

      // Get the position.
      let x = i * width,
        y = j * height;

      // Draw an axis...
      if( i === j ) {
        if( isDrawingAll ) {
          Axis.draw( x, y, width, height, canvas, nData, i );
        }
      }

      // ...or a plot.
      else {
        if( isFirstDraw ) {
          if( Matrix.bitmaps[ i ] === undefined ) {
            Matrix.bitmaps[ i ] = [];
          }
          Matrix.bitmaps[ i ][ j ] =
            Plot.draw( x, y, width, height, i, j, Matrix.scaled, canvas, opacity, Data.selectedRows );
        } else {
          Plot.draw( x, y, width, height, i, j, Matrix.scaled, canvas, opacity, Data.selectedRows, Matrix.bitmaps[ i ][ j ]);
        }
      }
    }
  }
};

// Axis.
const Axis = new Object();

/**
 * Draws the axis.
 *
 * @param  {number}  x        X coordinate, in pixels
 * @param  {number}  y        Y coordinate, in pixels
 * @param  {number}  width    width, in pixels
 * @param  {number}  height   height, in pixels
 * @param  {Element} canvas   CANVAS element
 * @param  {number}  nData    number of data values
 * @param  {number}  index    column index
 */
Axis.draw = ( x, y, width, height, canvas, nData, index ) => {
  
  // Initialization.
  let g = canvas.getContext( "2d" ),
    columnNames = Data.getColumnNames();
      
  // Draw the column label.
  g.fillStyle = "#000000";
  g.font = "14px Verdana";
  let s = columnNames[ index ];
  g.fillText( s, x + width / 2 - g.measureText( s ).width / 2, y + height - height / 2 + 4 );
  
  // Draw the minimum and maximum.
  g.font = "12px Verdana";
  s = "" + Math.round( 10 * Data.getDomain( nData, index )[ 0 ]) / 10;
  g.fillText( s, x + 4, y + height - 4 );
  s = "" + Math.round( 10 * Data.getDomain( nData, index )[ 1 ]) / 10;
  g.fillText( s, x + width - 3 - g.measureText( s ).width, y + 12 );
};

// Plot.
const Plot = new Object();

/**
 * Padding, in pixels.
 *
 * @constant {number}
 */
Plot.padding = 10;

/**
 * Cached bitmap, or none iff undefined.
 *
 * @constant {ImageData|undefined}
 */
Plot.imageData = undefined;

/**
 * Returns normalized rectangle.
 *
 * @param   {Rect}  rect   rectangle
 * @return  {Rect}  normalized rectangle
 */
Plot.normalize = ( rect ) => {
  let nx = rect.x,
    ny = rect.y,
    nw = rect.width,
    nh = rect.height;
  if( nw < 0 ) {
    nx += nw;
    nw = -nw;
  }
  if( nh < 0 ) {
    ny += nh;
    nh = -nh;
  }
  return { x: nx, y: ny, width: nw, height: nh };
}

/**
 * Returns whether point is within rectangle, within tolerance.
 *
 * @param  {Point}   point  point
 * @param  {Rect}    rect   rectangle
 * @param  {number}  tol    tolerance, or 0 for undefined
 */
Plot.isWithin = ( point, rect, tol ) => {
  let nRect = Plot.normalize( rect );
  if( tol !== undefined ) {
    nRect.x -= tol;
    nRect.y -= tol;
    nRect.width += 2 * tol;
    nRect.height += 2 * tol;
  }
  return ( nRect.x <= point.x ) && ( point.x < nRect.x + nRect.width  ) &&
         ( nRect.y <= point.y ) && ( point.y < nRect.y + nRect.height );
}

/**
 * Draws the plot.
 *
 * @param  {number}               x             X coordinate, in pixels
 * @param  {number}               y             Y coordinate, in pixels
 * @param  {number}               width         width, in pixels
 * @param  {number}               height        height, in pixels
 * @param  {number}               i             X column index
 * @param  {number}               j             Y column index
 * @param  {number[][]}           scaled        scaled coordinates
 * @param  {Element}              canvas        CANVAS element
 * @param  {number}               opacity       alpha
 * @param  {number[]}             selectedRows  indices of selected rows
 * @param  {ImageData|undefined}  imageData     bitmap of deselected points, or undefined if none
 * @return {ImageData}            bitmap of deselected points
 */
Plot.draw = ( x, y, width, height, i, j, scaled, canvas, opacity, selectedRows, imageData ) => {
    
  // Initialization.
  const g = canvas.getContext( "2d" ),
    padding = Plot.padding,
    scaledi = scaled[ i ],
    scaledj = scaled[ j ],
    nRows = scaledi.length,
    data = Data.getValues( nRows ),
    nBytes = width * height * 4;
  let deselectedImageData = imageData;
      
  // Create the deselected bitmap if necessary.
  // For alpha blending, see e.g. https://en.wikipedia.org/wiki/Alpha_compositing#Alpha_blending.
  if( deselectedImageData === undefined ) {
    deselectedImageData = g.createImageData( width, height );                           // black and transparent
    const d = deselectedImageData.data;
    data.forEach(( datum, row ) => {
      let xScaled = scaledi[ row ],
        yScaled = height - scaledj[ row ],
        k = ( yScaled * width + xScaled ) * 4;
      if(( 0 <= k ) && ( k + 3 < nBytes )) {
        d[ k     ] = Math.round(             0 + d[ k     ] * ( 1 - opacity ));     // r
        d[ k + 1 ] = Math.round(             0 + d[ k + 1 ] * ( 1 - opacity ));     // g
        d[ k + 2 ] = Math.round(             0 + d[ k + 2 ] * ( 1 - opacity ));     // b
        d[ k + 3 ] = Math.round( 255 * opacity + d[ k + 3 ] * ( 1 - opacity ));     // alpha
      }
    });
  }
  
  // Copy the deselected bitmap.  Caching minimizes use of createImageData().
  if( !Plot.imageData || ( Plot.imageData.data.length !== nBytes )) {
    Plot.imageData = g.createImageData( width, height );                                // black and transparent
  } else {
    Plot.imageData.data.fill( 0, 0, nBytes );                                           // black and transparent
  }
  const myImageData = Plot.imageData;
  const d = myImageData.data;
  d.set( deselectedImageData.data );
  
  // Add the selected rows as specified.
  // Selected rows use opacity, but not alpha blending, in order to keep them bright.  TODO:  Explore alternatives to this.
  for( let m = 0; ( m < selectedRows.length ); m++ ) {
    let row = selectedRows[ m ],
      xScaled = scaledi[ row ],
      yScaled = height - scaledj[ row ],
      k = ( yScaled * width + xScaled ) * 4;
    if(( 0 <= k ) && ( k + 3 < nBytes )) {
      d[ k ] = Math.round( 255 + d[ k ] * ( 1 - opacity ));                           // r
    }
  }
  
  // Draw and return the bitmap.
  g.putImageData( myImageData, x, y, padding, padding, width - 2 * padding, height - 2 * padding );
  return deselectedImageData;
};

/**
 * Selects rows within the brush and returns them.
 *
 * @param  {number}     x       X coordinate, in pixels
 * @param  {number}     y       Y coordinate, in pixels
 * @param  {number}     width   width, in pixels
 * @param  {number}     height  height, in pixels
 * @param  {number}     i       X column index
 * @param  {number}     j       Y column index
 * @param  {number[][]} scaled  scaled coordinates
 * @param  {Rect}       brush   brush
 * @return {number[]}   indices of selected rows
 */
Plot.select = ( x, y, width, height, i, j, scaled, brush ) => {
    
  // Initialization.
  let selectedRows = [];
  const scaledi = scaled[ i ],
    scaledj = scaled[ j ],
    nRows = scaledi.length,
    xMin = Math.floor( Math.min( brush.x, brush.x + brush.width ) - x ),
    xMax = Math.floor( Math.max( brush.x, brush.x + brush.width ) - x ),
    yMin = height - Math.floor( Math.max( brush.y, brush.y + brush.height ) - y ),
    yMax = height - Math.floor( Math.min( brush.y, brush.y + brush.height ) - y );
  
  // Collect the selected row indices and return them.
  for( let row = 0; ( row < nRows ); row++ ) {
    let xScaled = scaledi[ row ],
      yScaled = scaledj[ row ];
    if(( xMin <= xScaled ) && ( xScaled < xMax ) && ( yMin < yScaled ) && ( yScaled <= yMax )) {
      selectedRows.push( row );
    }
  };
  return selectedRows;
};
```

```js
//
// I set up a new Google Analytics property, 4E65ZQQL9N, but neither JS nor HTML work as is.
// JS require does not work locally. There is a d3.require package, but it's not recommended.
//
// Google tag (gtag.js)
// pageAnalytics = {
// await require(`https://www.googletagmanager.com/gtag/js?id=G-4E65ZQQL9N`).catch(() => {});

import { resolveFrom, requireFrom } from 'd3-require';
const myRequire = requireFrom( resolveFrom( `https://www.googletagmanager.com/gtag/js?id=G-4E65ZQQL9N/` ));
myRequire( "data" ).then(
   const dataLayer = window.dataLayer = window.dataLayer || [];
   const location = html`<a href>`;
   gtag('js', new Date());
   gtag('config', 'G-4E65ZQQL9N', {
     'page_path': location.pathname,
     'page_location' : location.href
   });
   return md`<sup>[Google Analytics](https://beta.observablehq.com/@peter-hartmann/page-analytics)</sup>`;
   function gtag() { dataLayer.push(arguments); }
);
// }
```
```html
<!-- Google tag (gtag.js)>
<script async src="https://www.googletagmanager.com/gtag/js?id=G-4E65ZQQL9N"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-4E65ZQQL9N');
</script -->
```
