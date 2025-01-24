---
layout: post
author: kormang
---


To understand how async functions work in JavaScript we will have to familiarize ourselves with few concepts. We will not go to the level of IO loop, because it is built into JavaScript engine and it would complicate things unnecessarily. In the next post we will build our own event loop in Python which can support async functions in Python.

## Coroutines, promises, generators, and iterators

Promises are objects that are supposed to hold a value that may not be ready at the moment, but can be ready in the future. That is why in some programming languages, a Promise is called a Future, Task, or Deferred. Future may be the best name, as it is short for "future value". For example, we might initiate an HTTP request to get some value, some results from a remote server. The function that initiates the request may send the request but return immediately instead of waiting for the remote server to return the result. Instead of the result, the function can return a Promise, or Future, an object that will hold the result sometime in the future when the server responds. These objects may or may not have a property like `ready` to check if the real result is ready and the value is inside the promise object, or it can have callbacks to which one can subscribe and get notified when the result is ready. So it is, in fact, a variation of the event-based approach to concurrent IO.

Iterators are design patterns that allow us to iterate over some finite or infinite collection. In certain programming languages, they have a special role and are a built-in mechanism.

Generators are usually syntactic sugar for functions that return iterators.

Iterators, generators, and coroutines are very much related concepts in programming and are present in many programming languages. In some languages, they are more related than in others. In the next few posts, we will see how they are implemented and related to each other in a few programming languages, to get a better overview of the subject. Then we will derive some conclusions about cooperative multitasking and see what coroutines have to offer us, that callbacks and polling on one side, and threads on the other side, can't.


## Coroutines in JavaScript

Promises in JavaScript can be implemented like this (note that this is a simplified promise that doesn't do error handling and probably has a few other inconsistencies with the standard Promise).

```javascript
class Promise {
  _subscribers = [];

  resolver = (value) => {
    for (let s of this._subscribers) {
      s(value);
    }
  };

  then = (callback) => {
    return new Promise((resolve) => {
      this._subscribers.push((value) => {
        if (value && typeof value.then === "function") {
          value.then((newValue) => {
            resolve(callback(newValue));
          });
        } else {
          resolve(callback(value));
        }
      });
    });
  };

  constructor(executor) {
    executor(this.resolver);
  }
}

const promise = new Promise(resolve => {
  // Do something, then resolve promise when everything is ready.
  resolve("Return value");
})

// Subscribe to result.

```

For more details on Promises in JavaScript, [read MDN docs](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).

Classes in JavaScript are also implemented via functions and objects and are just syntactic sugar, but that is not important for us right now.

In JavaScript, an iterator is an object that implements the Iterator protocol, which means it has a `next` method, which returns another object with the attributes `value` (the value the iterator produces at this step), and `done` (whether the iteration is over).

An iterable is an object that has a `[Symbol.iterator]` property, which is a function returning an iterator. It can be used in `for (let i of iterable)` loops.

We can use iterables and iterators like this:

```javascript
const arr = [1, 5, 3, 2, 6]
for (element of arr) {
  console.log(element)
}
```

This is because arrays are iterables in JavaScript. Let's explore some non-standard iterator examples.

```javascript
function createIterator() {
  return {
    state: 0,
    next: function (input) {
      switch (this.state) {
        case 0:
          this.state++;
          return { value: "one", done: false };
        case 1:
          this.state++;
          if (input) {
            return { value: "two", done: false };
          }
        case 2:
          this.state++;
          return { value: "three", done: false };
        case 3:
          this.state++;
          return { value: "four", done: true };
        case 4:
          return { value: undefined, done: true };
      }
    },
  };
}

let iterator = createIterator();

console.log(iterator.next());
console.log(iterator.next(true));
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());

console.log("Now, let's use it through iterable.");

const iterable = {
  [Symbol.iterator]: createIterator,
};

for (let j of iterable) {
  console.log(j);
}

```

It produces the following output:

```
{ value: 'one', done: false }
{ value: 'two', done: false }
{ value: 'three', done: false }
{ value: 'four', done: true }
{ value: undefined, done: true }
Now, let's use it through iterable.
one
three
```

For more details on iterators and iterables, see the [docs on MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols).


The above code can be simplified with generators (functions with special `*` syntax, that return iterators).

```javascript

// this is generator
function* createIterator() {
  const input = yield "one";
  if (input) {
    yield "two";
  }
  yield "three";
  return "four";
}

let iterator = createIterator();

console.log(iterator.next());
console.log(iterator.next(true));
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());

console.log("Now, let's use it through iterable.");

const iterable = {
  [Symbol.iterator]: createIterator,
};

for (let j of iterable) {
  console.log(j);
}

```

This code produces the same output as the above code. Generators save us from manually managing state machine state, again.

Notice that a generator can accept an input argument that is returned by the yield statement. In a pure iterator implementation, it is a normal input argument.

Now, let's see some code that uses Promises.

```javascript
function sleep(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

function fetchMs() {
  return new Promise((resolve) => {
    // We use set timeout here to simulate real fetch, that might
    // make HTTP request to the server, and the server may return
    // the result "2000" after 1 second.
    setTimeout(() => {
      resolve(2000);
    }, 1000);
  });
}


function func() {
  const promise = fetchMs();
  // We can do something else while the promise is ready.
  // We will subscribe to event via `then` function, and the
  // callback will be called when we receive the result.
  promise.then(ms => {
    // This callback is called when we have successfully fetched
    // milliseconds from the "server".
    console.log("Will sleep now");
    sleep(ms).then(() => {
      console.log("Good morning");
      return "Return value";
    })
  });
}

func().then(v => console.log(v));
```

Since Promises in JavaScript allow chaining we can avoid nesting callback hell like this:

```javascript
function func() {
  const promise = fetchMs();
  // We can do something else while the promise is ready.
  // We will subscribe to event via `then` function, and the
  // callback will be called when we receive the result.
  promise
    .then((ms) => {
      // This callback is called when we have successfully fetched
      // milliseconds from the "server".
      console.log("Will sleep now");
      return sleep(ms);
    })
    .then(() => {
      console.log("Good morning");
      return "Return value";
    });
}
```

This is slightly better.

It is possible to make it much better by using generators. Generators of promises, actually. We will write function `MakeAsyncCoroutine` that takes generator, that generates promises. Inside this generator we will be able to `await` on promise by `yield`ing (or generating) that promise. So literally we will be generating promises, but the code we write will also look like as we are waiting for promise to be fulfilled. Let's see it, and think about it.

```javascript

function MakeAsyncCoroutine(generator) {
  return function () {
    const iteratorOverPromises = generator();
    function runNextThen(value) {
      const itRes = iteratorOverPromises.next(value);
      if (itRes.done) {
        return itRes.value;
      }
      if (itRes.value && typeof itRes.value.then === "function") {
        const promise = itRes.value;
        return promise.then(runNextThen);
      } else {
        return itRes.value;
      }
    }
    return runNextThen();
  };
}

const func = MakeAsyncCoroutine(function* () {
  const ms = yield fetchMs();
  console.log("Will sleep now");
  yield sleep(ms);
  console.log("Good morning");
  return "Return value";
});

func().then((v) => console.log(v));
```

Finally it can be fully simplified using async/await syntax.

```javascript
async function func() {
  const ms = await fetchMs();
  console.log("Will sleep now");
  await sleep(ms);
  console.log("Good morning");
  return "Return value";
}

func().then(v => console.log(v));
```

Now we know, that async functions in JavaScript are just syntactic sugar for something like `MakeAsyncCoroutine` and generators.

The previous short `async function func()` is syntactic sugar for the following code:

```javascript
class Promise {
  _subscribers = [];

  resolver = (value) => {
    for (let s of this._subscribers) {
      s(value);
    }
  };

  then = (callback) => {
    return new Promise((resolve) => {
      this._subscribers.push((value) => {
        if (value && typeof value.then === "function") {
          value.then((newValue) => {
            resolve(callback(newValue));
          });
        } else {
          resolve(callback(value));
        }
      });
    });
  };

  constructor(executor) {
    executor(this.resolver);
  }
}

function sleep(ms) {
  return new Promise((resolve) => {
    setTimeout(resolve, ms);
  });
}

function fetchMs() {
  return new Promise((resolve) => {
    // We use set timeout here to simulate real fetch, that might
    // make HTTP request to the server, and the server may return
    // the result "2000" after 1 second.
    setTimeout(() => {
      resolve(2000);
    }, 1000);
  });
}

function MakeAsyncCoroutine(generator) {
  return function () {
    const iteratorOverPromises = generator();
    function runNextThen(value) {
      const itRes = iteratorOverPromises.next(value);
      if (itRes.done) {
        return itRes.value;
      }
      if (itRes.value && typeof itRes.value.then === "function") {
        const promise = itRes.value;
        return promise.then(runNextThen);
      } else {
        return itRes.value;
      }
    }
    return runNextThen();
  };
}

const func = MakeAsyncCoroutine(function () {
  return {
    state: 0,
    next: function (ms) {
      switch (this.state) {
        case 0:
          this.state++;
          return { value: fetchMs(), done: false };
        case 1:
          this.state++;
          console.log("Will sleep now");
          return { value: sleep(ms), done: false };
        case 2:
          this.state++;
          console.log("Good morning");
          return { value: "Return value", done: true };
      }
    },
  };
});

func().then((v) => console.log(v));
```

This code produces exactly the same output and takes same amount of time to execute, as the short async function above.

We have it all unwrapped - async function into `MakeAsyncCoroutine` + generator of Promises, promises into their simplified implementation, and generators into manual generator using state machine and regular function that manages its state and returns objects with `value` and `done`. All unwrapped to good old JavaScript prior to ECMAScript 6 (ES2015).

> Notice that we're going along the happy path here, and don't do any error handling, to make things easier to understand. However, easier error handling is the most important that async functions give us to improve software reliability.

As you have already noticed, async functions in JavaScript always return Promises. Promises are awaitables. So these too are the same:

```javascript
const result = await asyncFunc(x, y);
```

```javascript
const promise = asyncFunc(x, y);
const result = await promise;
```

That also implies that we can call async function and not await on its resulting promise. The code will still be executed but nothing will process the result.

## Summary

* Async functions in JavaScript are wrappers around generators that manually do deferred iteration over promises the generator generates.
* Generators are just syntactic sugar to simplify writing iterators that have internal state machine that remembers last instruction it has executed.
* Promises in JavaScript are better way, but still just a wrapper around **callbacks**. Those callbacks are run by **event loop** that pulls events from the event queue.

> Note that real implementation of Promises and async functions is much more complicated, with many more details, and more optimized, and written in C++.

Next, let's see how async functions work in Python, and how to build our own event loop that supports async functions.
