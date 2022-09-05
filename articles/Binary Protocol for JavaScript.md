# Binary Protocol for JavaScript

Having a native support for JSON is one of the joys of developing a full-stack JavaScript application. JSON is simple, schema-less, and human-readable — qualities especially useful in the early stages of development when our data model is still prone to changes. This flexibility, though, comes at a cost of runtime overhead in size and processing.

JSON, being a text-based format, encodes all values as UTF-8 resulting in size overhead when dealing with non-textual data. It being schema-free means we have to encode the structure of our data models (e.g. object keys) alongside the data. We also do extra work when processing since we have to convert values to their text representation before encoding and decode binary back to text before parsing it as JSON.

Admittedly, this overhead is not an issue for an average web application; and it is alleviated by using compression and the fact that JavaScript engines are good at parsing JSON. There are, however, cases where the overhead is problematic and compression, for example, is inefficient and might increase the size of a message, such as exchanging small messages when collecting telemetry, exchanging data in real-time applications, or sending notifications. To address these limitations of JSON, we can employ a binary format.

## Binary Formats

There is a plethora of binary formats among established data-serialization formats[^1] with a variety of properties. To address the bespoken limitations of JSON, we have to focus on two properties: a) whether they are schema-based and b) support zero-copy operations.

Schema-less binary formats, such as MessagePack or FlexBuffers, offer a size reduction when compared to JSON. The main advantage of these formats is that they can be used as a drop-in replacement for JSON with minimal work. However, a) we are still encoding the structure with the data, thus, have a significant overhead, b) we cannot do zero-copy operations. With schema-based formats, like Protocol Buffers, FlatBuffers, or Cap’n Proto, we can avoid encoding structural information in messages, although some overhead is present in the form of offset pointers.

Zero-copy operations in this context mean our ability to look up parts of a message without copying the data or decoding the whole message. For example, on the server, we can check the important parts of a request first without parsing the whole request body, on the client — parse and render large responses partially to minimize our FCP[^2] time. This means a reduction in processing time by orders of magnitude for certain cases. Among the data-serialization formats, Cap’n Proto and FlatBuffers support zero-copy operations, while Protocol Buffers, JSON, and schema-less formats do not. However, both Cap’n Proto and FlatBuffers prioritize memory access speeds over message size resulting in size overhead due to padding used for data alignment.

## Raw Buffers with View

To achieve minimal size with zero-copy access we can employ so-called raw buffers. For example, to encode a JavaScript object, we can calculate the required size for each of its fields, arrange them sequentially to derive the layout schema, and use that to encode the fields in an ArrayBuffer. The resulting buffer has “raw” data without any information about its structure. We can decode it whole or access individual fields using the layout schema since it is common to all objects of its type. We will require pointers for optional and variable-length fields, but the overhead is still significantly less than from key-encoding or data alignment.

[Structurae’s View](https://github.com/zandaqo/structurae#binary-protocol) interface does exactly that: given a JSON Schema of an object (or array, or any supported type), View will calculate the layout schema to store it as a raw buffer and create a class to handle the buffer that extends DataView:

```javascript
import { View } from "structurae";

type Animal = { name: string; age: number; }

const AnimalView = View.create<Animal>({
  $id: "Animal",
  type: "object",
  properties: {
    name: { type: "string", maxLength: 10 },
    age: { type: "number", btype: "uint8" },
  },
});

const animal = AnimalView.from({ name: "Gaspode", age: 10 });
animal instanceof DataView; //=> true
animal.byteLength; //=> 14
animal.get("age"); //=> 10
animal.set("age", 20);
animal.toJSON(); //=> { name: "Gaspode", age: 20 }
```

The View uses JSON Schema for schema definition. In addition to being familiar to developers, using JSON Schema also allows the reuse of the schemas in other tools like JSON validators serving as a single source of truth. The schemas and all derived classes are strongly typed to leverage type hinting and intellisense in IDEs. Unlike other popular schema-based formats, View does not require pre-compiling — the layouts are calculated once upon initialization. In general, View aims to fit nicely within the JSON-centric architectures and development workflows of modern web applications.

It is important to note, that the goal of View is not to replace JSON throughout a full-stack JavaScript application. But to maximize performance in parts of the application that can benefit from drastically reduced message sizes and avoiding extra parsing steps by using zero-copy operations. We can use it in all the cases where JSON falls short: number-heavy data that would benefit from binary encoding, high-rate exchange of messages in real-time applications or between worker processes, or partial processing of messages for validation.

## A Full Stack Example

To showcase the View structures in action, we will use a familiar message board example; it is not the best example for the use of binary, but more apt examples would be domain-specific and hard to follow.

Imagine that you are maintaining a message board for your local chapter of the League Of Temperance[^3]. Let’s define our message structure:

```javascript
class BoardMessage {
    id: number;
    authorId: number;
    threadId: number;
    created: number;
    text: string;
    ...
}

const MessageView = View.create({
    $id: 'BoardMessage',
    type: 'object',
    properties: {
        id: { type: 'integer' },
        authorId: { type: 'integer' },
        threadId: { type: 'integer' },
        created: { type: 'number' },
        text: { type: 'string', maxLength: 1000 }
    }
}, BoardMessage);
```

Now, on the client, we can encode our messages using the `MessageView` class. Since it is a DataView, we can straight up send it as a request body using the Fetch API:

```javascript
const message = new BoardMessage();

window.fetch('/messages', {
    method: 'POST',
    body: MessageView.from({ id: 1, authorId: 26, threadId: 1035, created: Date.now(), text: 'not one drop!'}),
    headers: {
        'content-type': 'application/octet-stream',
    },
});
```

On the server, for example, Node using Express.js, we can receive the message as a buffer and operate on it without parsing or copying the request body:

```javascript
app.post('/messages', (req, res) => {
    const view = new MessageView(req.body.buffer, req.body.byteOffset, req.body.length);
    ...
})
```

Now, for example, we can do access-control and check if the message author can post in the thread and reject unauthorized requests without parsing the whole request body:

```javascript
...
const view = new MessageView(req.body.buffer, req.body.byteOffset, req.body.length);
const hasAuthorization = getAuthrorization(view.get('authorId'), view.get('threadId'));
if (!hasAuthorization) return res.sendStatus(403);
...
```

Reading a number from a buffer is orders of magnitude faster than parsing JSON, not to mention reduced load on GC in the long run.

In this example, we can go further. The View uses a special class [StringView](https://github.com/zandaqo/structurae#strings) to handle UTF-8 encoded strings, the class extends DataView with several string-related methods that can operate on encoded strings without decoding them into JavaScript strings. Let’s say we decided to introduce a bit of censorship and block messages containing “zer b-vord”:

```javascript
const bVord = new TextEncoder().encode('blood');

app.post('/messages', (req, res) => {
    const view = new MessageView(req.body.buffer, req.body.byteOffset, req.body.length);
    ...
    const textView = view.getView('text');
    if (textView.includes(bVord)) return res.sendStatus(400);
    ...
})
```

With `view.getView`, we are instantiating a DataView (a StringView in this case) over a part of the buffer without decoding the part, then using a method to search inside the encoded data. Again, we are saving on parsing invalid requests both in terms of speed and GC pressure.

Finally, we can decode our message into a JavaScript object. In this case, into a `BoardMessage` instance since we provided the constructor class upon creating the view class:

```javascript
...
const message = view.toJSON();
...
```

View also takes advantage of the hidden class optimization[^4] used by JavaScript engines: instead of creating an empty object `{}` and populating it with decoded fields, the View either uses a provided constructor (as in the example above) or generates code for such a constructor so that each new object is instantiated with the same number and order of fields. This leads to faster operations on objects down the line as well as reduced GC pressure.

Thus, with reduced message sizes, zero-copy zero-parsing checks, and serialization optimizations, we have highly improved the message handling capabilities of our board. Let those ne’er-do-well werewolves try to spam us now!

---

[^1]: [Comparison of data-serialization formats](https://en.wikipedia.org/wiki/Comparison_of_data-serialization_formats)
[^2]: [First Contentful Paint](https://developer.mozilla.org/en-US/docs/Glossary/First_contentful_paint)
[^3]: [The Discworld’s League of Temperance](https://wiki.lspace.org/Überwald_League_of_Temperance)
[^4]: [V8 Hidden class](https://engineering.linecorp.com/en/blog/v8-hidden-class/)