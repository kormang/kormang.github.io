---
layout: post
author: kormang
---

It might not seem relevant to our asynchronicity and concurrency series, but reactive programming is actually one of the ways to approach asynchronicity and concurrency.

The idea of reactive programming is to be concurrency agnostic, and we will soon see what that means.

## Introduction to Reactive Programming

Wikipedia page on [reactive programming](https://en.wikipedia.org/wiki/Reactive_programming) defines it like this:

> In computing, reactive programming is a declarative programming paradigm concerned with data streams and the propagation of change.

Further it says:

> With this paradigm, it's possible to express static (e.g., arrays) or dynamic (e.g., event emitters) data streams with ease, and also communicate that an inferred dependency within the associated execution model exists, which facilitates the automatic propagation of the changed data flow.

So, it is a declarative way of programming (telling the program what to do, not how, as much as possible), and it helps work with data streams.

It is similar to working with the Streams API in Java or using functional processing of arrays in JavaScript. Let's see an example.

```javascript
const total = orders
  .filter(o => o.type === "goods")
  .map(o => o.price * o.amount)
  .reduce((a, b) => a + b, 0)
```

First we filter orders, then we multiply price by amount and sum it all up.

Now let's see similar example, but without array of orders, instead orders are streamed over the websocket, and as the total amount is updated with each event it should update state of some label (span) in the DOM. Also to avoid to frequent updates of the DOM we want it debounced, to avoid updates more frequent then 2 seconds.

```javascript
// Assuming we have a span with id 'totalLabel' in our HTML
const totalLabel = document.getElementById('totalLabel');

let total = 0;
let debounceTimer;

// Function to update the total in the DOM
function updateTotalLabel(newTotal) {
  totalLabel.textContent = `Total: ${newTotal}`;
}

// Function to handle the incoming order data
function processOrder(order) {
  if (order.type === "goods") {
    total += order.price * order.amount;
  }

  // Debounce the DOM update
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => updateTotalLabel(total), 2000);
}

// WebSocket connection
const socket = new WebSocket('ws://example.com/orders');

// Event listener for incoming messages
socket.onmessage = function(event) {
  const order = JSON.parse(event.data);
  processOrder(order);
};

// Handling WebSocket errors (basic example)
socket.onerror = function(error) {
  console.log(`WebSocket Error: ${error}`);
};

```

We can process those web socket events as stream of event using Reactive Extensions library for JavaScript (RxJs).

```javascript
import { fromEvent } from 'rxjs';
import { filter, map, scan, debounceTime } from 'rxjs/operators';

// Assuming we have a span with id 'totalLabel' in our HTML
const totalLabel = document.getElementById('totalLabel');

// Function to update the total in the DOM
const updateTotalLabel = (total) => {
  totalLabel.textContent = `Total: ${total}`;
};

// Establishing WebSocket connection
const socket = new WebSocket('ws://example.com/orders');

// Creating an observable from WebSocket messages
fromEvent(socket, 'message')
  .pipe(
    map(event => JSON.parse(event.data)), // Parsing the incoming data
    filter(order => order.type === "goods"), // Filtering orders of type 'goods'
    map(order => order.price * order.amount), // Calculating total for each order
    scan((acc, val) => acc + val, 0), // Accumulating the total
    debounceTime(2000) // Debouncing the updates for 2 seconds
  )
  .subscribe(total => {
    updateTotalLabel(total);
  });

// Handling WebSocket errors
fromEvent(socket, 'error').subscribe(error => {
  console.log(`WebSocket Error: ${error}`);
});

// Handling WebSocket connection close
fromEvent(socket, 'close').subscribe(event => {
  if (event.wasClean) {
    console.log(`Connection closed cleanly, code=${event.code}, reason=${event.reason}`);
  } else {
    console.log('Connection died');
  }
});

```

This is declarative, simple, and concise. RxJS also takes care to make error handling easy enough. Not shown here, but Reactive Extensions libraries have developed complex mechanisms to deal with back pressure when events arrive faster than they could be processed. All this makes new reactive operators (like `map` and `scan`) difficult to implement but easy to use.

Reactive Extensions exist for many other languages. Here is an example of how it could look in Java.

```java
import io.reactivex.rxjava3.core.Observable;
import io.reactivex.rxjava3.schedulers.Schedulers;

// Assuming WebSocketClient is a class that handles WebSocket connections
WebSocketClient webSocketClient = new WebSocketClient();

Observable<Order> orderStream = webSocketClient.getOrderStream();

orderStream
    .observeOn(Schedulers.io()) // Ensuring operations happen on another thread
    .filter(order -> "goods".equals(order.type))
    .map(order -> order.price * order.amount)
    .scan(0.0, (acc, price) -> acc + price)
    // Switching to a single-threaded scheduler for the next operation
    .observeOn(Schedulers.single())
    .debounce(2, TimeUnit.SECONDS)
    .subscribe(total -> updateTotalLabel(total));

```

As we can see, RxJava also enables us to decide on which thread (or anything else) the code should execute. That makes RxJava **concurrency agnostic**. It is very flexible when it comes to where the code will be executed and treats **concurrency as an implementation detail**.

For use cases that are **data flow-oriented**, this is the best tool we have so far. However, there are events that are not so easy to process this way. Consider the example we had previously in the post [Layers of abstraction](./2025/01/12/AC-part2-layers-of-abstraction.html#application-using-event-handlers), where we had to process keypress events. This is an example where control flow is more important than data flow. On each new key, the control flow changes, and to process it using Reactive Extensions, we would either have to use functions that imitate control flow primitives (`if`, `while`, `switch`, etc.) and/or use more or less explicit state machines, and we have already shown how unpleasant that can be. The reason is that Rx, although as declarative as possible, is still based on callbacks.

An alternative approach (when it is more about **control flow**, and not as much **data flow**) is structured concurrency, which we will talk about more in the next post. To read more on structured concurrency vs reactive programming, a recommended read is this article - [Reactive's Looming Doom](https://www.javacodegeeks.com/2022/09/reactives-looming-doom-part-i-evolution.html).
