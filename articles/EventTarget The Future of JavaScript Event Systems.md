# EventTarget: The Future of JavaScript Event Systems

A small feature recently made its way into browsers, and it may change the way we structure our web applications: DOM’s EventTarget interface got itself a constructor.

Events and event handling are central to JavaScript. We have DOM events in browsers, EventEmitter in Node.js, and almost every JavaScript framework has its own event or messaging system.

The event system in Node.js is based on the EventEmitter class. One can extend EventEmitter and turn objects into event targets that can emit events, be listened to, and so forth. This was not the case in browsers with DOM events. Until recently, there was no way to make a custom JavaScript object a DOM event target and use methods such as `addEventListener` on it. What’s more, due to historical discrepancies in implementations of DOM, the JavaScript community got used to the idea of handling DOM with thick gloves of jQuery. Thus, instead of relying on DOM events in browsers we had to make our own, parallel, event systems for our frameworks, such as [Backbone.Events](http://backbonejs.org/#Events), [Ember‘s Evented](https://www.emberjs.com/api/ember/release/classes/Evented), and the likes.

## EventTarget

This is going to change with the introduction of the [EventTarget Constructor](https://dom.spec.whatwg.org/#ref-for-concept-event-dispatch②②). EventTarget is the interface implemented by all elements of DOM that act as event targets. It is DOM’s “EventEmitter” if you wish, it provides methods for emitting events with`dispatchEvent`; subscribing to events using `addEventListener`; and unsubscribing with `removeEventListener`. Much like EventEmitter in Node.js, we can now extend EventTarget and make our objects event targets in the browser:

```javascript
class MyTarget extends EventTarget {}
const target = new MyTarget();
target.addEventListener('move', () => console.log('target moved'));
target.dispatchEvent(new Event('move'));
//=> target moved
```

For those who want to supply additional data with your events, we have [CustomEvent](https://dom.spec.whatwg.org/#customevent) interface that allows just that:

```javascript
class MyTarget extends EventTarget {}
const target = new MyTarget();target.addEventListener(
    'move',
    (event) => console.log(`moved ${event.detail.steps} steps`)
);target.dispatchEvent(
    new CustomEvent('move', { detail: { steps: 2 }  })
);//=> moved 2 steps
```

## Limitations

There is a number of convenience features we got used to in our own event systems that are missing in DOM:

1. There is no way to list or look-up listeners (or handlers) to an event on a target. This is made worse by the fact that one has to specify exactly what listener he wishes to remove with `removeEventListener` and cannot, for example, remove all listeners of a target. Although modern browsers do not leak event listeners and they are supposed to be collected by GC once the targets are unreachable, some of us still prefer to do it manually. There is [a proposal](https://github.com/whatwg/dom/issues/412) to add `getEventListeners` method to the EventTarget interface, but so far no consensus is reached.
2. Event listeners are invoked in the context of their targets, that is, with `this` referring to the target. The only way to change that is to use arrow functions or `Function.prototype.bind`.
3. Browsers are adamant on calling EventTarget methods in the context of EventTarget instances. EventTarget cannot be used as a mixin and only instances of classes extending EventTarget can invoke its methods. Since JavaScript doesn’t have the multiple inheritance, we cannot simultaneously extend built-in classes such as Array or Map with EventTarget. Sometimes this context limitation surfaces in unexpected places. For example, we couldn’t call `dispatchEvent` from a proxy of an EventTarget instance:

```javascript
class MyTarget extends EventTarget {}
const target = new MyTarget();
const proxy = new Proxy(target, {});
proxy.dispatchEvent(new Event('event'));
//=> Uncaught TypeError: Illegal invocation
```

There are also few differences to keep in mind when comparing to “native” events fired by DOM:

1. Custom EventTargets do not participate in a tree structure. That is, we cannot define their parents or children to “bubble” over events. The DOM standard mentions a possibility of adding this feature later and I [opened an issue](https://github.com/whatwg/dom/issues/583) requesting it, but so far no one else seems interested.
2. Since we use `dispatchEvent` method to fire our events on custom EventTargets, all listeners are invoked synchronously unlike “native” events that are dispatched by the browser calling their listeners asynchronously.

## Browser Support

The EventTarget constructor is already available in Chrome 64 and it’s coming to Firefox 59. There are open tickets in [Edge](https://wpdev.uservoice.com/forums/257854-microsoft-edge-developer/suggestions/20015521-eventtarget-constructor) and [Safari](https://bugs.webkit.org/show_bug.cgi?id=67312) bug trackers to support this feature. The feature is in the living standard, and hopefully, it will be supported by all evergreen browsers soon.

For older browsers, there are polyfills that use custom event systems to mimic EventTarget, such as [event-target](https://www.npmjs.com/package/event-target) and [event-target-shim](https://www.npmjs.com/package/event-target-shim).