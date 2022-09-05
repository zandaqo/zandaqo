# An Exercise in Shortening Ids

Sometimes solving a simple problem can lead to interesting places...

On the Web Platform, we constantly deal with numerical and binary labels or identifiers (ids), and we often use them in URIs for our API endpoints or filenames. A couple of issues arise in the process:

1. Sequential ids simplify id discovery and can lead to a vulnerability known as "insecure direct object reference"[^1] (if access control is lacking). They also reveal the size and velocity of data they identify, leading to business intelligence leaks.
2. Randomized ids like UUID or MongoDB's ObjectId are better. Although, they are 2 to 4 times larger than sequential ids for storage, and their default hexadecimal encoding is suboptimal for URI, i.e. result in longer than necessary strings.

A popular solution to exposing ids is to store a separate randomized id as a "public" id alongside a "private" sequential one. There are two main issues with this approach:

1. We have to store and index a significantly larger extra field.
2. Architecture-wise, it violates the separation of concerns by mixing an application-specific concern with our domain entities.

What if we could encode our existing ids in such a way as to:

1. Obfuscate and make them reasonably hard to predict;
2. Shorten as much as URI permits;
3. Do it faster than the alternatives?

Below is a short overview of one such solution implemented by [objectid64](https://github.com/zandaqo/objectid64).

## Base64

Our requirement to have a URI compatible string limits us to the following set of 66 characters[^2]: `A-Z a-z 0-9 - _ . ~`. Naturally, we will round the number down to the closest power of two leaving us with 64 characters.

Using an off-the-shelf Base64 encoder won't obfuscate much. We can, however, write our own encoder with a configurable alphabet, where changing the arrangement of characters produces different encodings:

```javascript
import { ObjectId64 } from "objectid64";
// with the standard alphabet
const userId = new ObjectId64(); 
userId.fromInt(861996) //=>"DScs"

// with a custom alphabet
const orderId = new ObjectId64("0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ-_");
orderId.fromInt(861996) //=> "3isI"
```

It should be noted, that this is not encryption: it merely obfuscates and shortens ids enough to dissuade amateurs. Use encryption when ids need to be truly hidden, though, mind its performance impact.

## Bit Twiddling

To encode numerical ids (i.e. numbers) in Base64, we can use the classic base conversion algorithm with division and modulo operations. However, since our base is a power of two (64 = 2<sup>6</sup>), we can speed up the implementation quite a bit (2 to 5 times faster encoding in synthetic benchmarks) by using bitwise shift instead of division and bitwise AND instead modulo[^3] :

```typescript
  function toBase64(integer: number): string {
    let encoded = "";
    for (let n = integer; n > 0; n >>= 6) {
      encoded = this.base[n & 63] + encoded;
    }
    return encoded;
  }
```

In the same way, we can handle the 64-bit sequential ids by using JavaScript's now built-in `bigint` type.

## UUID Shortening

Hexadecimal encoding of UUID and ObjectId used in the wild is suboptimal for our use case of encoding for URI where we can use Base64 that is 25% more efficient[^4] (i.e. produce shorter strings). While we can convert from Base16 to Base64 using the base conversion algorithm above, we can do much better by creating a lookup table.

Notice that Base16 uses 16 characters that represent 4 bits each (2<sup>4</sup> = 16), whereas Base64 has 64 characters encoding 6 bits (2<sup>6</sup> = 64). Thus, three hexadecimal characters can encode 12 bits, the same as two characters of Base64. Since the amount of hex triplets is limited to 16<sup>3</sup> = 64<sup>2</sup> = 4096, we can create a reasonable lookup table linking each triplet to a pair of equivalent Base64 characters and use it to convert from one base to another without any calculations.

By converting from hex to Base64, we shorten ObjectId from 24 characters to 16, and UUID from 36 to 22:

```typescript
const encoder = new ObjectId64();
encoder.fromObjectId("581653766c5dbc10f0aceb55") //=> "WBZTdmxdvBDwrOtV"
encoder.fromUUID("6d2bb408-3176-42d3-b473-3d251f19569f") //=> "bSu0CDF2QtO0cz0lHxlWCf"
```

And do so faster than any alternatives, albeit by trading a bit of memory for the speed.

Of course, if we get the ids in their binary forms, we can forego hex encoding altogether and directly encode into our Base64:

```javascript
const encoder = new ObjectId64();
const objectId = new ObjectId().id;
//=> Uint8Array(12) [88,  22, ... 235,  85]
const encoded = encoder.fromBinObjectId(objectId.id);
//=> 'WBZTdmxdvBDwrOtV'
const decoded = encoder.toBinObjectId(encoded)
//=> //=> Uint8Array(12) [88,  22, ... 235,  85]
```

------

Thus, a simple task of making ids URL-friendly led us to contemplating architecture, learning in-depth about encodings and base conversions, employing bit twiddling hacks and memoization. In the end, your choice of ids should not be dictated by the way they look in URIs or even by the security concerns due to their exposure—you can always encode (or encrypt) into a preferred shape.

---

[^1]:[Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
[^2]: [RFC 4648](https://datatracker.ietf.org/doc/html/rfc4648)
[^3]: [Bit Twiddling Hacks](http://graphics.stanford.edu/~seander/bithacks.html#ModulusDivisionEasy)
[^4]: [Binary-to-text encoding](https://en.wikipedia.org/wiki/Binary-to-text_encoding#Encoding_plain_text)
