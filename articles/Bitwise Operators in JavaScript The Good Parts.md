# Bitwise Operators in JavaScript: The Good Parts

Bitwise operators, for the most part, have been shunned by the JavaScript community. Douglas Crockford labels them as a “bad part” in his seminal book “JavaScript: The Good Parts”. Popular style guides suggest avoiding them where possible. The few voices raised in their support every now and then usually point out their performance advantages in certain scenarios or the few symbols they save compared to alternatives. In a world of ever so smarter engines and transpilers, those advantages sound less and less convincing, though.

That all said, I would like to share a few examples where I find the bitwise operators useful for everyday programming in JavaScript. I assume the reader is familiar with the basic workings of the bitwise operators, otherwise please refer to the [documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators).

## The Bad

There are two main reasons mentioned when bitwise operators are deemed “bad”:

1. Bitwise operators can make code hard to reason about for a human reader. The bitwise `&` and `|` can be confused with logical `&&` and `||`.
2. Numbers in JavaScript are represented by 64-bit values, but bitwise operators always return a 32-bit integer. This leads to unexpected behavior for integer values larger than 32 bits.

These are valid concerns that should be kept in mind when one considers using bitwise operators. In the old days, bitwise operations were also slow due to the overhead of casting 64-bit values to and from 32-bit integers. This no longer applies to modern JIT compilers that are happy to keep numbers as integers unless casting them to a floating point value is warranted.

## The Good

There is [a long list](http://graphics.stanford.edu/~seander/bithacks.html) of things one can do with bitwise operators. Shorter versions of that list often appear with recommendations somewhat specific to JavaScript. But I’ll have to agree with skeptics that most of those things are very niche and/or better done by other means in JavaScript… with two exceptions:

### Integer casting with `|0`

`x|0` (as well as `~~x`, `x|x`, etc.) can be used to convert anything to an integer much like `parseInt` and `Math.trunc()`. Unlike its alternatives, `x|0` never produces `NaN` and works [much](https://jsperf.com/math-floor-vs-math-round-vs-parseint/18) faster:

```javascript
“3.0” | 0 //=> 3
{} | 0 //=> 0
parseInt({}, 10) //=> NaN
Math.trunc({}) //=> NaN
```

Although, unlike `parseInt` it’s not safe for numbers larger than 32 bits:

```javascript
“2147483649” | 0 //=> -2147483647
parseInt(“2147483648”, 10) //=> 2147483648
```

This technique is popular enough that ESLint's `no-bitwise` rule has an option `int32Hint` that makes an exception for `x|0` pattern.

### Checking for -1 with ~

Both `Array#indexOf` and `Array#findIndex` return -1 if an element is absent from an array. To account for the absence, one can always compare the result with `index === -1`, yet I find using the bitwise not `~` operator much more succinct and readable:

```javascript
!!~1 //=> true
!!~0 //=> true
!!~-1 //=> false
!!~-2 //=> trueconst index = array.indexOf(element);
if (~index) {
 // element exists
}
```

## The Beautiful

[Bit fields](https://en.wikipedia.org/wiki/Bit_field) are perhaps the most popular use case for bitwise operators outside of low-level languages. In JavaScript, we can treat a number as a bitfield that can store up to 31 boolean values and all we need is a couple lines of code:

```javascript
class Bits {
  static get(field, index) {
    return (field >> index) & 1;
  }
  static set(field, index, value = 1) {
    return (field & ~(1 << index)) | (value << index);
  }
}
```

To give an example, let’s say we have a web application with notifications where user can turn on and off different types of notifications. We might be tempted to implement it as an array of booleans indicating whether the user is subscribed to a particular type or not:

```javascript
[
 true, // 0 - notify if there is a new comment
 true, // 1 - a new article
 false, // 2 - do not notify if there is a new like 
]
```

However, we can save space (and traffic) by “packing” these booleans into a single number and treating its bits as booleans:

```
let field = parseInt(‘011’, 2); //=> 3
Bits.get(field, 0) //=> 1 // new comment notification
Bits.get(field, 1) //=> 1 // new article
Bits.get(field, 2) //=> 0 // new like// we can change bits as well
field = Bits.set(field, 0, 0)
Bits.get(field, 0) //=> 0; // no notifications for new comments
Bits.get(field, 1) //=> 1
```

Those comfortable (and brave enough) with bitwise operators can forego using these functions and employ the full power of the operators:

```javascript
// we can store masks instead of indexes
const notified = {
  comment: 1,
  article: 1 << 1,
  like: 1 << 2,
};// and use them directly
// to set a bit:
let field = 0 | notified.comment;// to set another bit:
field = field | notified.article;// to check a bit
!!(field & notified.comment) //=> true// to check multiple bits at once
!!(field == (field | (notified.comment | notified.article)))
//=> true
```

What’s more, popular databases such as PostegreSQL and MongoDB also support bitwise operators. That allows us to store our bitfield as a single number and query each bit separately if needed. In case of MongoDB, we have `$bitsAllSet` query operator and `$bit` update operator for that:

```javascript
// get all users who want to be notified if a new article came out
// (index 1) 
db.collection(‘users’).find({ field: { $bitsAllSet: [1] })// to subscribe a user to like notifications (index 2):
db.collection(‘users’).update({id: 10},
  { $bit: { field: { or: (1 << 2) } } })// to unsubscribe
db.collection(‘users’).update({id: 10},
  { $bit: { field: { and: ~(1 << 2) } } })
```

Thus, we don’t have to “unpack” or convert our bitfield at any layer of our application, from database to client we can keep it as a single number.

## It gets better

Integers are coming to JavaScript with the [BigInt](https://github.com/tc39/proposal-bigint) proposal. Bitwise operations on BigInts return a BigInt that is not limited to 32 bits. Hence, we can use bitwise operators on integers of any size. As well as using BigInts as bitfields of unlimited size. The proposal is currently at Stage 3 and likely to land in ECMAScript 2019. BigInts are already implemented in V8 and available in Chrome and Node.js.