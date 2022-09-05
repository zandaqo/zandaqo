# JavaScript ♥ C++: Modern Ways to Use C++ in JavaScript Projects

Using C++ code in JavaScript projects is nothing new. Node.js has supported [C](https://nodejs.org/dist/latest-v8.x/docs/api/addons.html)[++ Addons](https://nodejs.org/dist/latest-v8.x/docs/api/addons.html) since the beginning. In browsers, [asm.js](http://asmjs.org/) has been around for years now. Yet there had been issues with both approaches that made the experience less than perfect.

The classic Node.js C++ Addon API exposes the underlying components of Node.js such as V8 and libuv which makes it powerful yet fragile since changes in the components can break the addons relying on them. Developers have to recompile and often update their C++ addons with every new version of Node.js. To solve this and other issues, Node.js is introducing new [N-API](https://nodejs.org/dist/latest-v8.x/docs/api/n-api.html) that is independent from the underlying JavaScript runtime (e.g. V8) and promises a stable binary interface (ABI) across Node.js versions.

Asm.js is a subset of JavaScript that can be optimized to reach near-native performance and serve as a compilation target for languages such as C++. While it is officially supported in Firefox and Edge, and being JavaScript it is well-treated by others, it is set to be replaced by [WebAssembly](http://webassembly.org/). For all its benefits, asm.js is still JavaScript and has some of the same limitations, for example, it still has to be delivered as text and parsed the same way. WebAssembly, on the other hand, offers a binary format that allows for a smaller size and [much faster parsing](https://hacks.mozilla.org/2017/03/why-webassembly-is-faster-than-asm-js/).

## Trying N-API and WebAssembly

I have created `iswasmfast`project to try both N-API and WebAssembly in Node.js. The idea is to use off-the-shelf C++ implementations of various algorithms (*.cpp files in `lib` folder) integrating them into Node.js as an N-API Addon and a WebAssembly module in the most straight forward way. The performance of the resulting C++ Addon, the WebAssembly module, and native implementations in JavaScript (*.js files in `lib`) is compared in a benchmark.

## N-API

Node.js 8 offers new API for C++ Addons called N-API. It is still experimental and can be enabled with `— napi-modules`command-line flag. As per description, it offers higher-level abstractions and stability across versions.

N-API is C API and one can notice that it can get quite verbose in more complicated cases. In `iswasmfast`, I’ve opted for the [node-addon-api](https://github.com/nodejs/node-addon-api) package that offers C++ wrapper classes making N-API a lot more concise and palatable. The resulting glue code that allows calling the C++ functions from JavaScript is located in `src/node.cpp`. N-API and node-addon-api are well-documented, so I won’t go into details here. In general, though, it’s pretty straightforward: we include all the C++ functions we need; write a wrapper for each function that converts received JavaScript arguments to C++ values to use them in our C++ code, and converts the result back to a JavaScript values; finally, we expose all the wrapper functions to JavaScript in the `Init` method.

## WebAssembly

The first, MVP, version of WebAssembly is already in Node.js, Chrome, Firefox, and soon to come to Edge 16 and Safari 11. [A lot is written](https://github.com/mbasso/awesome-wasm) about WebAssembly already. While much talk is about how WebAssembly is going to dislodge JavaScript in browsers, I’d like to mention few important but overlooked things it brings to the JavaScript community. First, not only can we now use compiled languages in browsers, we can use the same code in Node.js sharing it between the browser and server much like JavaScript. Second, in some cases, WebAssembly modules can prove more performant in Node.js than C++ Addons due to the penalties associated with moving data in and out of JavaScript engines.

Right now, the go-to way to compile C++ code into a WebAssembly module is to use [Emscripten SDK](https://github.com/juj/emsdk). The setup and compilation steps are [well-documented](http://webassembly.org/getting-started/developers-guide/) on the WebAssembly homepage. Emscripten offers various ways to connect C++ and JavaScript. One can [call C++ functions](http://kripken.github.io/emscripten-site/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#interacting-with-code-ccall-cwrap) directly (with or without ccall and cwrap) if only primitive values are exchanged. More involved cases require special bindings created with [Embind](http://kripken.github.io/emscripten-site/docs/porting/connecting_cpp_and_javascript/embind.html) or [WebIDL](http://kripken.github.io/emscripten-site/docs/porting/connecting_cpp_and_javascript/WebIDL-Binder.html). WebIDL is well-suited for exposing larger interfaces with minimal effort. Embind, on the other hand, allows both direct calls when using primitive values and defining wrappers when passing objects.

In `iswasmfast` I went with Embind. You can see the connecting code in `src/wasm.cpp`. As with N-API, we import all the functions and expose them to JavaScript, this time using `EMSCRIPTEN_BINDINGS` macros. Unlike N-API, though, we don’t need to write wrappers for functions exchanging primitive values: numbers, booleans, or strings. The only wrapper function we have is for the simple linear regression that has to accept JavaScript arrays and convert them to `std::vector` and back. There is no additional configuration and we just compile it with: `emcc — bind -std=c++14 src/wasm.cpp -s WASM=1 -O3 -o wasm.js`. This results in generating two files: `wasm.wasm` that contains the WebAssembly binary, and `wasm.js` with the glue code that simplifies loading and working with the binary.

## Benchmark

```
> node --napi-modules benchmark.js

Levenstein Distance:
   Native x 102,775 ops/sec ±1.12% (87 runs sampled)
   N-API Addon x 209,164 ops/sec ±0.64% (90 runs sampled)
   Web Assembly x 145,086 ops/sec ±1.15% (90 runs sampled)
 Fastest is N-API Addon

Fibonacci:
   Native x 3,776,692 ops/sec ±0.83% (86 runs sampled)
   N-API Addon x 3,329,576 ops/sec ±1.06% (90 runs sampled)
   Web Assembly x 8,420,121 ops/sec ±0.82% (89 runs sampled)
 Fastest is Web Assembly

Fermat Primality Test:
   Native x 1,820,068 ops/sec ±0.90% (90 runs sampled)
   N-API Addon x 1,710,430 ops/sec ±0.66% (89 runs sampled)
   Web Assembly x 2,781,497 ops/sec ±0.65% (88 runs sampled)
 Fastest is Web Assembly

Simple Linear Regression:
   Native x 21,680 ops/sec ±2.19% (89 runs sampled)
   N-API Addon x 4,024 ops/sec ±1.23% (90 runs sampled)
   N-API Addon using TypedArrays x 15,094 ops/sec ±0.86% (85 runs sampled)
   Web Assembly x 1,687 ops/sec ±0.67% (90 runs sampled)
 Fastest is Native

SHA256:
   Native x 6,018 ops/sec ±1.28% (87 runs sampled)
   N-API Addon x 12,243 ops/sec ±1.39% (85 runs sampled)
   Web Assembly x 9,267 ops/sec ±0.72% (88 runs sampled)
 Fastest is N-API Addon
```

The performance of the N-API addon is hampered by the conversions allowing for native implementations to be almost on par with it and WebAssembly to outperform it in most examples. As one would expect, N-API addon shines in computing the Levenstein distance and SHA256 hashes where relatively less data is exchanged while still being computationally heavy.

WebAssembly mostly outperforms native implementations as one would expect. Except for the linear regression example where no doubt the conversion from and to arrays took a toll on performance. There are ways make it much more performant using TypedArrays and [direct memory access](http://kapadia.github.io/emscripten/2013/09/13/emscripten-pointers-and-pointers.html), but it’s a bit more involved and I hope for a better solution to land in Emscripten.

Surprisingly, WebAssembly manages to be twice as fast as N-API in computing Fibonacci numbers and testing primality with the Fermat method. It might be the case where transferring data between WebAssembly and JavaScript suffers less penalty than passing it in and out of the JavaScript engine when using N-API Addon.

## Conclusions

With N-API and WebAssembly, using C++ code in JavaScript becomes easier than ever and requires very little knowledge of C++. N-API Addons might prove to be not only easier to develop, but more importantly easier to support, bringing more C++ into the Node.js ecosystem. WebAssembly is just making its first steps, but it’s already delivering on its performance promises. It also allows us to share the same C++ code between Node.js servers and browsers. In Node.js, WebAssembly might also give an extra edge over C++ Addons in tasks that require a frequent exchange of data with JavaScript.