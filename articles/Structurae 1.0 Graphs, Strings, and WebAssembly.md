# Structurae 1.0: Graphs, Strings, and WebAssembly

[Structurae](https://github.com/zandaqo/structurae), a library of data structures for high-performance JavaScript introduced here[^1] previously, gets a stable release adding memory efficient graphs, strings as byte streams, sorted structures, and more to its growing collection. Here I’d like to introduce some of the new structures and touch on a few related subjects.

## Notes on Binary in JavaScripts

Although dealing with binary was not the focus of structurae, it had to go “low” in its quest for efficiency. Under the hood, structurae makes heavy use of TypedArrays and bitwise operators to act on byte and bit level. This is not as common as one would expect, so let us reiterate a few basics of dealing with binary in JavaScript.

The go-to way of representing binary data in JavaScript is ArrayBuffers, usually operated through TypedArrays or DataView interfaces. Unless you are dealing with bits in which case you can get away with Numbers and bitwise operators (e.g. structurae’s `BitField`). This might seem obvious, but I see a lot of JavaScript libraries employing strings, Node.js Buffers, or even Arrays of ones and zeros for no good reason other than probably supporting IE 9.

Creating ArrayBuffers is comparatively expensive, and the intended way of using them is to share a single buffer between multiple TypedArrays (or DataViews) pointing to different regions of the buffer using offsets. For the same reason, it is better to use simple Arrays instead of TypedArrays for short-lived objects. Recently I have come across a seemingly popular library that would mimic C types by creating a whole new ArrayBuffer for each primitive value it gets... with predictably poor performance.

## Memory Efficient Graphs

JavaScript libraries implementing graphs seem to focus on either graph visualizations or serving as learning material for students. In both cases, the performance impact gets overlooked and the implementations are woefully inefficient when it comes to dealing with large graphs. For example, some implementations opt for representing each edge with a JavaScript object which is 40–104 bytes[^2], whereas for unweighted graphs a single bit would do. The exorbitance of it really shows once we start dealing with hundreds of thousands of nodes and edges.

To deal with large graphs efficiently, structurae offers classes implementing adjacency matrix and list for both weighted and unweighted graphs. Internally, the structures use ArrayBuffers to hold the adjacency information in the densest possible way. As an example, for an unweighted graph, the adjacency matrix uses a single bit in an underlying ArrayBuffer to denote an edge between two nodes. For weighted matrices, we use `Grid` variations that result in an “unrolled” TypedArray using a single ArrayBuffer. For sparse graphs, we have adjacency list implementations that also rely on TypedArrays and require less space, however, editing operations on lists are slower due to a possible shifting of values when edges are added or removed.

```javascript
const { WeightedAdjacencyMatrixMixin } = require('structurae');
// creates a class for directed graphs that uses Int32Array for edge weights
const WeightedAdjacencyMatrix = WeightedAdjacencyMatrixMixin(Int32Array, true);
const matrix = new WeightedAdjacencyMatrix({ vertices: 6, pad: -1 });
matrix.addEdge(0, 1, 5) // add edge between 1 and 5 vertices with a weight of 5
  .addEdge(0, 2, 1)
  .addEdge(0, 3, 6)
  .addEdge(2, 4, 3)
  .addEdge(2, 5, 2);
matrix.hasEdge(0, 1);
//=> true
matrix.hasEdge(0, 4);
//=> false
[...matrix.outEdges(2)];
//=> [4, 5]
[...matrix.inEdges(2)];
//=> [0]
```

It should be noted that these are not succinct data structures[^3] — we are simply using the most efficient structures that JavaScript offers to reduce our memory footprint. Although we do have `RankedBitArray` that is often used for implementing succinct structures.

The adjacency structures above serve as basic classes for implementing graphs. Structurae also has the `Graph` class that extends a given adjacency structure with common methods for graph traversal, pathfinding, tree creation, etc:

```javascript
const { GraphMixin, WeightedAdjacencyMatrixMixin }  = require('structurae');
// for directed weighted graphs that use adjacency matrix structure
const WeightedGraph = GraphMixin(WeightedAdjacencyMatrixMixin(Int32Array));
const graph = new WeightedGraph({ vertices: 6, edges: 12 });
graph.addEdge(0, 1, 3)
  .addEdge(0, 2, 2)
  .addEdge(0, 3, 1)
  .addEdge(2, 4, 8)
  .addEdge(2, 5, 6);
[...graph.traverse()]; // a BFS traversal results
//=> [0, 1, 2, 3, 4, 5]
[...graph.traverse(true)]; // DFS
//=> [0, 3, 2, 5, 4, 1]
// BFS yeilding only non-encountered ('white') vertices starting from 0
[...graph.traverse(false, 0, false, true)];
//=> [1, 2, 3, 4, 5]
graph.path(0, 5);
//=> [0, 2, 5]
```

## The Joy of C-like Strings

With Edge adopting Chromium, all modern browsers now support the Encoding API[^4]. The API essentially lets us convert JavaScript strings to a byte stream and back. We had libraries doing it before, but this API brings built-in support and, more importantly, establishes a common standard for representing strings in binary: it uses `Uint8Array` and encodes into UTF8.

To take further advantage of the API, structurae introduces `StringView` class that extends `Uint8Array` with common string related methods such as `search`, `replace`, `substring`, etc. The methods operate on the underlying byte stream directly avoiding the need to convert it back to a JavaScript string. The class also operates on Unicode characters, allowing, for example, iterating over the characters, unlike strings that use UTF16 code points.

```javascript
const { StringView } = require('structurae');
const stringView = StringView.fromString('abc😀a');
//=> StringView [ 97, 98, 99, 240, 159, 152, 128, 97 ]
stringView.toString();
//=> 'abc😀a'
stringView == 'abc😀a';
//=> true
stringView.length; // length of the view in bytes
//=> 9
stringView.size; // the amount of characters in the string
//=> 5
stringView.charAt(0); // get the first character in the string
//=> 'a'
stringView.charAt(3); // get the fourth character in the string
//=> '😀'
[...stringView.characters()] // iterate over characters
//=> ['a', 'b', 'c', '😀', 'a']
stringView.substring(0, 4);
//=> 'abc😀'
stringView.search(StringView.fromString('😀')); // equivalent of String#indexOf
//=> 3
stringView.replace(StringView.fromString('😀'), StringView.fromString('d')).toString();
//=> 'abcda'
stringView.reverse().toString();
//=> 'adcba'
```

Although `StringView` methods are usually slower than their counterparts for strings, avoiding conversions while dealing with binary data can boost the overall performance.

## WebAssembly Integration

The premise of structurae is getting JavaScript code reliably fast without resorting to micro-optimizations or rewriting it to target WebAssembly. That said, there are cases where we can benefit from offloading certain tasks to WebAssembly.

One thing that currently hinders the performance in such a scenario is copying and serializing data passing to and from WebAssembly. However, the raw memory of WebAssembly is exposed to JavaScript through the Memory[^5] object and can be operated on directly by JavaScript without copying. Since the Memory is a *resizable* ArrayBuffer, we can operate on it using the familiar typed arrays as well as all the data structures in structurae that extend typed arrays.

It gets better — Emscripten’s Embind[^6] and AssemblyScript’s loader[^7] modules both offer handy utility functions for creating an array in WebAssembly and returning pointers that can be used to create a corresponding typed array on the JavaScript side. In other words, we can use JavaScript and WebAssembly to operate on the same data without moving it around. Here is an example of doing it with AssemblyScript loader:

```javascript
const fs = require("fs");
const loader = require("assemblyscript/lib/loader");
const { GraphMixin, WeightedAdjacencyMatrixMixin }  = require('structurae');
const WeightedGraph = GraphMixin(WeightedAdjacencyMatrixMixin(Int32Array));
const myImports = {};
const myModule = loader.instantiateBuffer(
  fs.readFileSync(__dirname + "/build/optimized.wasm"),
  myImports
);
// create a Int32Array in the WebAssembly memory large enough 
// to store a graph of 128 vertices and return a pointer to it
const graphPointer = myModule.newArray(Int32Array, WegitedGraph.getLength(128), true);
// use the pointer to create a view over the WebAssembly memory
const graphView = myModule.getArray(Int32Array, graphPointer);
// use the view to create an instance of WeightedGraph in JavaScript
const graph = new WeightedGraph(
  { vertices: 128 },
  graphView.buffer,
  graphView.byteOffset,
  graphView.length
);
```

There is a catch, though, when WebAssembly’s Memory grows, the typed arrays we use as “views” into it become detached, i.e. out of sync with WebAssembly. The issue along with the proposed solution[^8] is currently actively discussed. For now, however, we will have to keep track of the memory by ourselves.

---

[^1]: [Structurae: Data Structures for High Performance JavaScript](https://blog.usejournal.com/structurae-data-structures-for-high-performance-javascript-9b7da4c73f8)
[^2]: [Understanding the size of an object in Chrome/V8](https://www.mattzeunert.com/2017/03/29/v8-object-size.html)
[^3]: [Succinct data structure](https://en.wikipedia.org/wiki/Succinct_data_structure)
[^4]: [Encoding API](https://developer.mozilla.org/en-US/docs/Web/API/Encoding_API)
[^5]: [WebAssembly.Memory](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory)
[^6]: [Emscripten’s Embind](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html)
[^7]: [AssemblyScript’s loader](https://github.com/AssemblyScript/assemblyscript/tree/master/lib/loader)
[^8]: [Proposal: Memory grow event / handler](https://github.com/WebAssembly/design/issues/1210)