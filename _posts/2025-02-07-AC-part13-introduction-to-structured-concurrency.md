---
layout: post
author: kormang
---

## Introduction to Structured Concurrency

### Example

Consider this example: we want to make an HTTP request, and if it doesn't complete in 2 seconds, we want to cancel it. We want to wrap that into a function called `fetchWithTimeout`, which returns `undefined` in case of a timeout and response data if everything was successful.

```javascript
async function fetchWithTimeout(resource, options = {}, timeout = 2000) {
  const controller = new AbortController();

  setTimeout(() => {
    statusLabel.innerText = "Canceling request due to timeout...";
    controller.abort();
  }, timeout);

  const response = await fetch(resource, {
    ...options,
    signal: controller.signal
  })

  return await response.json();
}

async function otherFunction(url) {
  const response = fetchWithTimeout(url);
  if (response === undefined) {
    statusLabel.innerText = "Canceled";
  }
  statusLabel.innerText = "Received data " + JSON.stringify(response);
}

```

This is really simple example, and the implementation is simple thanks too `async`/`await` syntax and finally block. However it has few bugs. In case of cancellation error will be thrown, so we need to catch it. Also, we need to clear timeout if request is completed in less then 2 seconds.

```javascript
async function fetchWithTimeout(resource, options = {}, timeout = 2000) {
  const controller = new AbortController();

  const id = setTimeout(() => {
    statusLabel.innerText = "Canceling request due to timeout...";
    controller.abort();
  }, timeout);

  try {
    const response = await fetch(resource, {
      ...options,
      signal: controller.signal
    })
    return response;
  } catch (e) {
    if (e.name === 'AbortError') {
      return undefined
    } else {
      throw e;
    }
  } finally {
    clearTimeout(id);
  }
}

async function otherFunction(url) {
  const response = fetchWithTimeout(url);
  if (response === undefined) {
    statusLabel.innerText = "Canceled";
  }
  statusLabel.innerText = "Received data " + JSON.stringify(response);
}
```

This is a simple example and a straightforward solution. However, it still requires a lot of manual work. For instance, we need to remember to put `clearTimeout` in the `finally` block, among other things.

In this case, we have two concurrent tasks: one is to fetch the data, and the other one is waiting for 2 seconds before aborting the fetch. Imagine that we have many concurrent tasks, some animations based on `setInterval` or `requestAnimationFrame`, some asynchronous functions waiting for input from the user, and so on. It would be challenging to cancel all of them when either 2 seconds elapse or the fetch completes.

To illustrate that, we will add another concurrent task: we will listen for a click event on a button. After the operation is completed or canceled, we will stop listening (in reality, we might want to hide or remove the button, but let's say that we just want to stop listening). If button is clicked before request is completed it will abort request.

```javascript
async function fetchWithTimeout(resource, cancelBtn, options = {}, timeout = 2000) {
  const controller = new AbortController();

  const onCancelBtnClick = () => {
    statusLabel.innerText = "Canceling request on user demand...";
    controller.abort();
  };

  cancelBtn.addEventListener("click", onCancelBtnClick);

  const id = setTimeout(() => {
    statusLabel.innerText = "Canceling request due to timeout...";
    controller.abort();
  }, timeout);

  try {
    const response = await fetch(resource, {
      ...options,
      signal: controller.signal
    })
    return response;
  } catch (e) {
    if (e.name === 'AbortError') {
      return undefined
    } else {
      throw e;
    }
  } finally {
    clearTimeout(id);
    cancelBtn.removeEventListener("click", onCancelBtnClick);
  }
}

async function otherFunction(url) {
  const response = fetchWithTimeout(url, document.getElementById("cancel-button"));
  if (response === undefined) {
    statusLabel.innerText = "Canceled";
  }
  statusLabel.innerText = "Received data " + JSON.stringify(response);
}
```

Similar logic implemented using RxJS would look something like this:

```javascript

import { fromEvent, race, timer, of } from 'rxjs';
import { takeUntil, tap, finalize } from 'rxjs/operators';
import { ajax } from 'rxjs/ajax';

const httpRequest$ = ajax.getJSON(URL);
const timeout$ = timer(2000);
const cancelBtn = document.getElementById('cancelBtn');

const cancel$ = fromEvent(cancelBtn, 'click');

const requestWithTimeout$ = race(httpRequest$, timeout$).pipe(
  takeUntil(cancel$),
  finalize(() => {
    // Cleanup: remove event listener and any other cleanup tasks
    cancelBtn.removeEventListener('click', onCancelClick);
  })
);

// Function to handle cancel button click
const onCancelClick = () => {
  console.log("Canceling request on user demand...");
};

// Add event listener for the cancel button
cancelBtn.addEventListener('click', onCancelClick);

// Subscribe to the composed observable
requestWithTimeout$.subscribe({
  next: response => {
    // Handle the response or timeout here
    if (response) {
      console.log('Response received:', response);
    } else {
      console.log('Request timed out');
    }
  },
  error: error => {
    // Handle any errors here
    console.error('Error:', error);
  },
  complete: () => {
    console.log('Operation completed or canceled');
  }
});

```

RxJS takes care about some things, we don't have to care about clearing timeout, and `takeUntil` will close the observable that emits click events.

But the fact that RxJS is based on callbacks, makes it [difficult to construct control flows](/2025/01/12/AC-part2-layers-of-abstraction.html#application-using-event-handlers) using this approach. Another drawback is that we need to known what each of the RxJS functions exactly does to understand whether the code will do what it is supposed to do.

While callbacks make it difficult to follow control flow, it is not the reason why there is reliability issues when we use callbacks. We would still have the problem even if we would have callback-free code using hypothetical async functions as follows:

_This is the same problem taken from another approach. Instead of callbacks we use waiting._

Instead of event listener we will have event waiter, and instead of timeout, we will have sleep.

```javascript
async function fetchWithTimeout(resource, cancelBtn, options = {}, timeout = 2000) {
  const controller = new AbortController();

  async function waitForCancelClick() {
    await cancelBtn.waitClickEvent(controller.signal);
    statusLabel.innerText = "Canceling request on user demand...";
    controller.abort();
  };

  async function sleepAndAbort() {
    await sleep(timeout, controller.signal);
    statusLabel.innerText = "Canceling request due to timeout...";
    controller.abort();
  };

  try {

    // Here is THE PROBLEM, we don't await for those two function,
    // so they can run concurrently, but if there is some error in any of
    // those, it will not be caught, it will (in case of JavaScript)
    // silently fail.
    waitForCancelClick();
    sleepAndAbort();

    const response = await fetch(resource, {
      ...options,
      signal: controller.signal
    })
    return response;
  } catch (e) {
    if (e.name === 'AbortError') {
      return undefined
    } else {
      throw e;
    }
  }
}

async function otherFunction(url) {
  const response = fetchWithTimeout(url, document.getElementById("cancel-button"));
  if (response === undefined) {
    statusLabel.innerText = "Canceled";
  }
  statusLabel.innerText = "Received data " + JSON.stringify(response);
}
```

APIs like this do not exist in JavaScript, but even if they do we would have the problem presented here. Executing async functions with await will not allow them to run concurrently, and without await any errors there will go unhandled. When it comes to cancellation passing abort signal to those functions can help, but it still requires to manually pass it.

Obviously, with callbacks the problem still remains. If we're using callbacks via RxJS, error handling mechanism and functions like `finalize` can help handle errors. However, it is still hard to see the **structure** of all the concurrent paths, both "happy" paths and error handling paths.

Solution to these problems is **structured concurrency**. It does not currently exist in JavaScript, but if it would, it would look something like the following code.

In this example we also have connection to websocket, that we are using to send some tracking/logging data. In this hypothetical example connection has to be short lived for some reason, we open it do few things, and we close immediately after that. This connection is shared between tasks, and needs to be closed when all tasks finish their execution, either regularly, or by being cancelled, or after error is thrown inside task; we need a way to guarantee that this resource will be cleaned.

```javascript
async function fetchWithTimeout(resource, cancelBtn, options = {}, timeout = 2000) {
  return startTaskScope(taskScope => {
    // Example of resource that is shared between tasks,
    // and has to be cleaned up at the end.
    const connection = await openNewWebsocketConnectionToTrackingService();

    async function waitForCancelClick() {
      await cancelBtn.waitClickEvent();
      statusLabel.innerText = "Canceling request on user demand...";
      await logCancelButtonClicked(connection);
      taskScope.cancel();
    };

    async function sleepAndAbort() {
      await sleep(timeout);
      statusLabel.innerText = "Canceling request due to timeout...";
      await logSleepCompleted(connection, timeout);
      taskScope.cancel();
    };

    try {
      // Start few concurrent tasks.
      taskScope.startSoon(waitForCancelClick);
      taskScope.startSoon(sleepAndAbort);

      // Continue with the main task of the scope.
      const response = await fetch(resource, options);
      await logResponse(connection, response);
      taskScope.cancel();
    } finally {
      // We can be sure that all tasks have finished by now,
      // and nobody is using this connection any more
      // (no dangling tasks running somewhere).
      // Now it is safe to close the connection.
      // There is only one place where we have to call close.
      // We don't have to think about all the possible ways
      // those tasks could exit.
      // Of course this assumes none of the tasks have stored reference to this
      // connection in some global variable or anything similarly stupid.
      connection.close();
    }

    return response;
  });
}

async function otherFunction(url) {
  try {
    const response = fetchWithTimeout(url, document.getElementById("cancel-button"));
    statusLabel.innerText = "Received data " + JSON.stringify(response);
  } catch (e) {
    statusLabel.innerText = "Canceled";
  }
}

```

If either of `fetch`, `waitForCancelClick`, `sleepAndAbort` throws and exception, other two will be canceled and the exception will be propagated to try-catch block in the calling function. When either of them exits normally, it will cancel other two in regular way by calling `taskScope.cancel()`.

In structured concurrency, calling `taskScope.cancel` is equivalent to `break` or early `return` statements in structured programming. It is supposed to cancel all the tasks in the scope by throwing some kind of `ScopeCancellationException` out of the async function that belongs to the runtime environment (JS engine). In this case, it can be thrown from `fetch`, `sleep`, or `waitClickEvent`. Throwing such an error should interrupt the execution of all tasks in the scope, and the `startTaskScope` should catch it so it doesn't propagate outside the scope. All finally blocks and error handlers will be properly executed, making it easy to reason about the reliability of the software. The concise principle of work behind it and the concise syntax make it easy both to write and read.

In contrast, with callback based mechanism, we could `setTimeout` or listen for a button click, which could continue running even if connection is closed, or it would be difficult to know when to actually close the connection. Similarly with tasks based concurrency, but without structured concurrency, we would have to manually ensure that we know when all tasks have finished, and when it is safe to close the connection.

Without structured concurrency, we could ensure that all tasks are finished like in this example - but structured concurrency is **forcing us to ensure there are no dangling tasks that continue to run** even after we exit the scope; which is quite regular practice in concurrent programming without structured concurrency.

Of course, all of this is just hypothetical; there is no such API in JavaScript. Given the over reliance on callbacks and async functions based on promises, which are based on `resolve` and `reject` callbacks, it is very difficult, if not impossible, to implement structured concurrency in JavaScript.

### The core idea

We already talked about [structured programming](/2024/08/22/evolution-of-code.html#structured-programming). Structured concurrency is the equivalent in the world of concurrent programming.

The core idea is that you can not just create concurrent task (either explicitly, or by expecting callback like `setTimeout`). Instead you create scopes, similarly to how `if` statement creates new scope. Concurrent tasks are started within scope, and at the end of the scope program waits for all tasks created inside scope to finish (either regularly, or due to exception, or by being cancelled). For example if tasks are threads, waiting for them to finish would be accomplished by calling `join`, but with few extra features - most importantly it has to be able to handle exceptions thrown in the thread.

This solves two very important and hard problems in concurrent programming - **error handling and cancellation**.

It is hard to track all the resources we need to clean up after tasks exits (e.g. `clearTimeout`, `abort`, closing files, streams, deallocating memory). When it exits in irregular way (being cancelled or error occurs inside the task) making sure all the resources are cleaned up is even more difficult. We simply have to think about too many ways tasks could exit. This is analogous to how `goto` statements could jump almost anywhere, making it difficult to perform cleanup operations, since we had no guarantees about order of execution. Structured programming introduced those guarantees, and structured concurrency introduces equivalent guarantees to concurrent programming - code that follows task scope will always execute after all tasks in the scope have finished, one way or another.

### Learn from the original creators

It is not too broad topic, but still there is a lot to talk about when it comes to structured concurrency. To get deeper understanding about why structured concurrency is necessary it is highly recommended to read the article [Notes on structured concurrency, or: Go statement considered harmful](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) by Nathaniel J. Smith, who came up with the term structured concurrency and implemented first such mechanism for Python language in the form of [trio](https://trio.readthedocs.io/en/stable/) library for async concurrency and I/O (it is later implemented in non-pure form in standard `asyncio` too).

There is also great [talk](https://www.youtube.com/watch?v=Mj5P47F6nJg) about structured concurrency by Roman Elizarov, whose team invented structured concurrency independently (the concept, not the term) while implementing coroutines for Kotlin language. There, basic motivation for structured concurrency is well explained, which is error handling and cancellation.

These two resources, especially combined, do a great job at explaining structured concurrency, much better then I could do.

Python and Kotlin are first two languages to adopt structured concurrency. Other languages have also started adopting structured concurrency, Swift (since 2021), Java (part of Project Loom, in Java 21), end there are some proposals to add it to Go too.
