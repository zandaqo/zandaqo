# Structurae: Data Structures for High Performance JavaScript

As the adage goes, JavaScript is not the best choice for CPU-intensive tasks. But sometimes it is a good enough choice. When it comes to serious number crunching, my first instinct is to hack together a microservice in C++ or stick a native addon to a Node.js API server. Yet often doing so brings diminishing returns in performance due to the data marshalling overhead. WebAssembly might remedy this to a degree[^1], but again, we might not need it as often as we think[^2].

After my last attempt to speed up a Node.js server by replacing some JavaScript with a native addon yielded less than exciting results, I have decided to collect and share some common structures I use to optimize my JavaScript code: [**structurae**](https://github.com/zandaqo/structurae).

Here I would like to describe the techniques used in the library and explore its potential use cases. I would like to emphasize that the techniques aim at performance-sensitive applications such as games or real-time data processing. They are hardly applicable to most of our run-of-the-mill Todo apps and might do more harm than good if used indiscreetly.

## The General Approach

In the good old days, performance optimizations in JavaScript meant avoiding certain language constructs that turned out slow in one engine or another. Nowadays JavaScript engines update so rapidly that it is pointless to chase the “performance killers”. We can, however, help the engines by addressing the “weak points” of JavaScript when it comes to performance: dynamic typing and garbage collection. The recommendations to that end can be summarized as follows:

1. Simplify and normalize data structures. Where possible, prefer simple types over objects, numbers over strings, small integers[^3] over other numbers, TypedArrays over Arrays, and dense Arrays over sparse Arrays[^4].
2. Be consistent with the “shapes” of objects. Set object properties and their default values upon creation, and avoid adding new properties or changing types of the values later on[^5].
3. Avoid creating garbage. Watch out for `new`, `[]`, `{}`, and methods creating new objects, e.g. `Array#slice`, `Array#splice` etc.

In [**structurae**](https://github.com/zandaqo/structurae) I am mostly focusing on the first item by providing a set of classes that aid in representing and operating on complex data structures using simple numbers and ArrayBuffers.

## Grid

Grid is a mixin class that extends Array or TypedArray class to handle 2-dimensional data. In other words, you can replace your nested arrays with a single array and still address items using two indexes aka coordinates. This by itself can boost performance in certain situations, especially with TypedArrays where it would result in using a single ArrayBuffer.
But Grid also attempts to speed up lookups. Normally, finding the index of an element in an “unrolled” array using its coordinates or vice versa would involve multiplications and divisions:

```javascript
const index = row * rowLength + column;
const row = Math.floor(index / rowLength);
const column = index  -  (row * rowLength);
```

Grid uses a much faster bitwise shift instead:

```javascript
const index = row << rowLength + column;
const row = index >> rowLength;
const column = index - (row << rowLength);
```

For this to work, our `rowLength` has to be a power of two (2, 4, 8…), so Grid will pad rows to the nearest power of two that can fit the row.

An obvious use case for Grid would be 2D data associated with maps in games, but I found it useful even in text processing. Performance impact may vary, but benchmarks usually show a 10–40% boost for simple operations when compared to nested structures.

```
Grid Get/Set:
 Nested arrays x 20,767,307 ops/sec ±9.04% (75 runs sampled)
 Grid x 29,498,717 ops/sec ±9.77% (72 runs sampled)
 Fastest is Grid
Grid Iterate:
 Nested arrays x 12,292 ops/sec ±10.48% (64 runs sampled)
 Grid x 15,310 ops/sec ±9.97% (66 runs sampled)
 Fastest is Grid
```

## BitField

Treating numbers as a set of bits representing boolean values is a well-known optimization technique[^6]. In JavaScript, we can do so using bitwise operators[^7]. By default, BitField class does exactly that: treats a number as a bitfield of 31 bits and provides methods to get, set, and check bits:

```javascript
const bitfield = new BitField(0b11101);
bitfield.get(0);
//=> 1
bitfield.get(1);
//=> 0
bitfield.has(2, 3, 4);
//=> true
bitfield.set(2, 0);
bitfield.value
//=> 20 === 0b11001
```

However, there is no reason to be limited by a single bit: we can pack as many integers into a number as we can fit within the size limit. For that, we can extend BitField and define our own schema specifying the desired width of each field in bits:

```javascript
class Person extends BitField {}
Person.fields = [
 { name: 'age', size: 7 },
 { name: 'gender', size: 1 },
];
const person = new Person([20, 1]);
person.get('age');
//=> 20
person.set('age', 18);
person.toObject();
//=> { age: 18, gender: 1 }
person.toValue();
//=> 146
```

If the total size of your fields exceeds 31 bits, BitField will try to use BigInts internally, but it will fall back to numbers if BigInts are not supported.

This kind of “packing” numbers has an obvious benefit of having less data to deal with. But I found it even more useful for fast pattern matching sets of numbers. To give you a real-life example, let us say we have texts with words represented as sets of labels, positive integers, describing their characteristics such as part of speech, gender, person, etc. If we want to filter words with certain labels, our naive implementation would look like this:

```javascript
const matchWord = (word, matcher) => {
 for (let i = 0; i < matcher.length; i++) {
   if (matcher[i] && (matcher[i] !== word[i])) return false;
 }
 return true;
};
const matcher = [1, 0, 2, 0, 4];
const matches = text.filter(word => matchWord(word, matcher);
```

In the worst case scenario, where all but the last label matches those in each word, this will perform **2 \* m \* n** comparisons, where **m** is the length of a matcher and **n** is the number of words.
With BitField we can do it in **n** comparisons:

```javascript
/ create a class to encode word arrays
class Word extends BitField {}
Word.fields = [
  { name: 1, size: 4 },
  { name: 2, size: 4 },
  { name: 3, size: 4 },
  { name: 4, size: 4 },
  { name: 5, size: 4 },
];
Word.initialize();
const matcher = {1: 1, 3: 2, 5: 4};
// encode the matcher using the same class, and create a mask to 
// "unset" irrelevant fields in the matching words
const encodedMatcher = Word.getMatcher(matcher);
const matches = text.filter(word => Word.match(word, encodedMatcher);

// where `Word.match` is just one bitwise operation and one comparison:
class BitField {
  //...
  static match(value, matcher) {
    return (value & matcher[1]) === matcher[0];
  }
  //...
}
```

In benchmarks BitField matching is about twice as fast as the naive implementation with arrays:

```
BitField Match:
 Native x 140,494 ops/sec ±9.82% (69 runs sampled)
 BitField x 299,072 ops/sec ±11.89% (76 runs sampled)
 Fastest is BitField
```

## RecordArray

RecordArray brings C-like structs or records[^8] by extending DataView and using ArrayBuffer as an array of records. That is, you can put an array of objects with fields of different types into a single ArrayBuffer. RecordArray supports all the numerical types supported by DataView plus strings.

```javascript
// create an array of 20 records where each has 'age', 'score', and 'name' fields
const people = new RecordArray([
 { name: 'age', type: 'Uint8' },
 { name: 'score', type: 'Float32' },
 { name: 'name', type: 'String', size: 10 },
], 20);
// get the 'age' field value for the first struct in the array
people.get(0, 'age');
//=> 0
// set the 'age' and 'score' field values for the first struct
people.set(0, 'age', 10).set(0, 'score', 5.0);
people.toObject(0);
//=> { age: 10, score: 5.0 }
```

A RecordArray stores all its data in a single ArrayBuffer. It might lead to a smaller memory footprint, but more importantly, raw binary data of ArrayBuffers has much less overhead for transfer and parsing when compared to JSON. Basically, RecordArrays can act as a binary protocol for efficient high-frequency data exchange over networks, e.g. exchanging user data in multiplayer games. Both Fetch and WebSocket APIs support ArrayBuffers making exchanging RecordArrays a breeze:

```javascript
const schema = [
 { name: 'age', type: 'Uint8' },
 { name: 'score', type: 'Float32' },
 { name: 'name', type: 'String', size: 10 },
];

const records = new RecordArray(schema, 20);

// to send a RecordArray
fetch('/records', { method: "POST", body: records })

// to receive a RecordArray
fetch('/records/1')
  .then(response => response.arrayBuffer())
  .then(buffer => {
    // create a RecordArray for the received ArrayBuffer
    // the size parameter is disregarded when buffer is supplied
    return new RecordArray(schema, 0, buffer);
  });
```

## Pool

In JavaScript, creating and destroying objects at a high-rate increases the pressure on the Garbage Collector and can have a noticeable impact on the overall performance. Object pooling[^9] is one the common techniques used in such cases: we use a set of pre-allocated objects — the pool — recycling them after each use instead of destroying.

A common way to implement object pooling is to have an array of objects where each object has a flag denoting if it is currently in use or not. Each time we need an object, we iterate over the array looking for the first available object. We can optimize this by having a separate TypedArray of the same length where each element acts as flag (1 or 0) showing availability of the corresponding object in the object pool. In that case, each time we need an object we can use the built-in `TypedArray#indexOf` to find next available object. The implementation might look like this:

```javascript
class NaivePool {
    constructor(size) {
      this.register = new Uint8Array(size).fill(0);
      this.nextAvailable = 0;
    }
    // find next available index
    get() {
      const { nextAvailable, register } = this;
      if (!~nextAvailable) return -1;
      const available = register[nextAvailable];
      // set the current object to unavailable setting it to 1
      register[nextAvailable] = 1;
      // find the next available by finding a 0
      this.nextAvailable = register.indexOf(0);
      return available;
    }
    // free the current index
    free(index) {
      this.register[index] = 0;
      this.nextAvailable = index;
    }
  }
```

We can do better, though, about 10 times better if benchmarks are to be believed. Instead of storing the flags in a separate TypedArray of the same length, we can store them as bits in bitfields, making the resulting TypedArray 16 times smaller. And instead of using `Typedrray#indexOf`, we look for the first set bit with a single bitwise operation that finds us the least significant bit in the field: `x &= -x`. That is exactly how the Pool class works under the hood with a bit of extra magic.

```javascript
// create a pool of 1600 indexes
const pool = new Pool(100 * 16);

// get the next available index and make it unavailable
pool.get();
//=> 0
pool.get();
//=> 1

// set index available
pool.free(0);
pool.get();
//=> 0

pool.get();
//=> 2
```

## Afterword

Hopefully, these will help you boost performance of your JavaScript code without obfuscating it with microoptimizations. In my experience, when the architecture and algorithms and are taken care of, optimizing few core data structures is usually enough to get the most out of an application.

[structurae](https://github.com/zandaqo/structurae) is a work in progress where I hope to collect more of the common structures I encounter in high-performance applications. Aside from the above mentioned classes, it also includes SortedArray and SortedCollection that extend built-in Array and TypedArrays respectively to efficiently handle sorted data, but that’s another story for another day.

---

[^1]: [JavaScript ♥ C++: Modern Ways to Use C++ in JavaScript Projects](https://medium.com/netscape/javascript-c-modern-ways-to-use-c-in-javascript-projects-a19003c5a9ff#f542)
[^2]: [Maybe you don’t need Rust and WASM to speed up your](https://mrale.ph/blog/2018/02/03/maybe-you-dont-need-rust-to-speed-up-your-js.html)
[^3]: [Performance Tips for JavaScript in V8](https://www.html5rocks.com/en/tutorials/speed/v8/#toc-topic-numbers)
[^4]: [Elements kinds in V8](https://v8.dev/blog/elements-kinds)
[^5]: [Javascript Hidden Classes and Inline Caching in V8](https://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html)
[^6]: [Bit field](https://en.wikipedia.org/wiki/Bit_field)
[^7]: [Bitwise Operators in JavaScript: The Good Parts](https://medium.com/@zandaqo/bitwise-operators-in-javascript-the-good-parts-a2e811a8c76b#e93f)
[^8]: [Record_(computer_science)](https://en.wikipedia.org/wiki/Record_(computer_science))
[^9]: [Object pool pattern](https://en.wikipedia.org/wiki/Object_pool_pattern)