---
source: https://nodejs.org/docs/latest/api/events.html#events
tags:
  - backend
  - nodejs
  - draft
  - node-docs
---
node is built on an asynchronous event-driven architecture  
emitter emit events causing functions called listeners to run

emitter are instances of the `EventEmitter` class.  
emitters expose the `on(eventName, listener)` method for registering listeners for a certain event  
`eventName` maybe any valid property key i.e `number | string | symbol`  
emitters expose the `emit(eventName, ...args)` method that causes all registered listeners to be run *synchronously*.
`...args` are passed directly into the invoked listeners.  
`this` in the invoked listeners references the emitter instance on which the event was triggered.  
if an arrow function listener is used then `this` points to the outer scope.

note: 
  `Function.prototype.call` does not work on arrow functions

```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter();

emitter.on("sneeze", function(this: Emitter, count: number, threshold: number){
  console.log("first");

  console.log(this === emitter);

  for(let i = 0; i < count; i++){
    console.log("achooo");

    if(i < threshold){
      console.log("bless you");
    } else {
      console.log("oya its enough");
      break;
    }

    console.log("---")
  }
});

emitter.on("sneeze", (count: number, threshold: number) => {
  console.log("second");

  console.log(this === global)
  if(5 < count) console.log("covid detected, nuking city");
  else console.log("no infection detected");
})

// all arguments are forwarder to listeners
// listeners are invoked in registration order
emitter.emit("sneeze", 5, 2);

// All handlers are synchronously called hence marker prints last
console.log("marker");

```

> [!todo]
> - implement `EventEmitter` in typescript. 
> - emphasis on the typing of the `on` and `emit` methods.


***async events***
listeners are called in registration order i.e. synchronously.  
`setImmediate` or `process.nextTick()` can be used within a listener schedule asynchronous work.
```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter();

emitter.on("sneeze", () => {
  console.log("first");

  setImmediate(() => {
    console.log("third");
    console.log("bless you one")
  })
})

emitter.emit("sneeze");
console.log("second")
```

> [!question]
> investigate the node event loop and where `setImmediate` and `process.nextTick()` fit in


***one hit wonder***
`emitter.once(eventName, listener)` can be used to register a listener that is invoked only then firs time `eventName` is emitted.  
note:
  `listener` is unregistered *and then* invoked.

```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter();

emitter.once("sneeze", function foobar(this: Emitter){
  // foobar already unregistered
  // hence listenerCount === 0
  console.log(emitter.listenerCount("sneeze"));
})

console.log(emitter.listenerCount("sneeze")); // 1

emitter.emit("sneeze");

emitter.emit("sneeze"); // nothing happens
```

***error events***
a `error` event is triggered when an error occurs within an `EventEmitter` instance.  

note:
  - this is by convention meaning implementers of the `EventEmitter` are expected to manually trigger the `error` event whenever they choose.
  - an `error` event is ***not*** automatically emitted when an error is thrown inside a listener.
  - differs from `Promise` which wraps errors automatically an passes then on to catch handlers

```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter();

emitter.on("error", ({ name }) => {
  console.log("Oh no", name);
})

emitter.on("event", function(){
  throw "unimplemented";
})

// error event not triggered
emitter.emit("event");
```

the error to be thrown is passed as an argument to `emitter.emit("error", err)`

```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter();

emitter.on("error", ({ message }) => {
  console.log("Oh no!", message);
})

emitter.emit("error", new Error("your mum"));
```

if there are no registered handlers the error is thrown and the process crashes.
the `errorMonitor` symbol can be used to monitor `error` events without consuming them

***capture rejection of promises***
if an `async` function is used as a listener for an event any error thrown within that function are routed to the global `unhandledRejection` listener. (as per usual)

```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter();

emitter.on("event", async function(){
  throw new Error("boom");
})

emitter.emit("event");

// unhandled rejection: boom
process.on("unhandledRejection", ({ message }) => {
  console.log("unhandled rejection:", message);
})
```

setting `captureRejections: true` in the config object argument of the `EventEmitter` constructor routes these error to the `Symbol.for('nodejs.rejection')`  if defined or the `error` event listeners.

```typescript
class Emitter extends EventEmitter {}

const emitter = new Emitter({ captureRejections: true });

// error listener: boom
emitter.on("error", ({ message }) => {
  console.log("error listener:", message);
})

emitter.on("event", async function(){
  throw new Error("boom");
})

emitter.emit("event");

// does not run
process.on("unhandledRejection", ({ message }) => {
  console.log("unhandled rejection:", message);
})
```

tip:
  this can be used to implement the `Promise` like behavior alluded to above.

note:
  setting `EventEmitter.captureRejections` sets the default for all new instances of `EventEmitter`


`EventEmitter` class 

***events***
- `'newListener'` 
    - `emitter.on('newListener', (eventName: string, listener: Listener) => void)`
    - `'newListener'` event is emitted ***before*** a `listener` is added to the internal listener array.
- `'removeListener'` 
    - `emitter.on('removeListener', (eventName: string, listener: Listener) => void)`
    - `removeListener` is emitted ***after*** `listener` is removed

***methods***
- `emit`
    - `(eventName: string, ...args: any[]) => boolean`
    - synchronously invokes all registered listeners for `eventName` in registration order.  
    - returns `true`/`false` depended on whether or not `eventName` had registered listeners.
- `eventNames`
    - `() => string[]`  
- `getMaxListeners`
    - `() => number`
- `listenerCount`
    - `(eventName: string, listener?: Listener) => number`
    - returns the number of registered listeners for `eventName`. 
    - If `listener` is provided it returns the number of times `listener` was registered for `eventName`
- `listeners`
    - `(eventName: string) => Listener[]`
    - returns a ***copy*** array of listeners for `eventName`
-  `(off | removeListener)`
    - `(eventName: string , listener: Listener) => EventEmitter`
    - removes most recent registration of `listener` from the internal listener array
    - Listeners is removed during the process of invoking the registered listeners when an event is emitted are still invoked regardless even though they have been removed from the array.
    - copies of the internal array become invalidated after calls to `off`
-  `(on | addListener)`
    - `(eventName: string, listener: Listener) => EventEmitter`  
    - appends `listener` to the array of listeners for `eventName`.  
- `prependListener`
    - `(eventName: string, listener: Listener) => EventEmitter`  
    - appends `listener` to the array of listeners for `eventName`.  
- `once`
    - `(event: string, listener: Listener) => EventEmitter`
    - registers a one time listener for `event`
    - `listener` is removed ***and then*** invoked
- `prependOnceListener`
    - `(event: string, listener: Listener) => EventEmitter`
- `removeAllListeners`
    - `(event?: string) => EventEmitter`
    - if `event` is omitted listeners for all events are removed
- `setMaxListeners`
    - `(n: number) => EventEmitter`
    - sets the maximum number of listeners that can be registered for all events before a warning is is printed
- `rawListeners`
    - `(event: string) => Listener[]`
    - returns a copy of the listeners array for `event` including any wrappers.

***static propeties***

![[Pasted image 20240208165101.png]]

![[Pasted image 20240208165119.png]]