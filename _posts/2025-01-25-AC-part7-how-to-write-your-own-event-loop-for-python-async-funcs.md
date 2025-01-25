---
layout: post
author: kormang
---


## Event Loop for coroutine tasks

[We saw what coroutines are under the hood.](/2025/01/25/AC-part6-how-python-coroutines-work.html).

Let's examine further how async/await works. We have the following function, named `func`. It calls the `fetch_ms` function, which simulates a call to a web server but will actually return when the user presses the enter key. Instead of waiting for the server to respond, we will wait for the user to respond, as it is simpler and easier for you to try out yourself, but the concept remains the same. Instead of waiting for the STDIN file descriptor to be ready for reading, real HTTP requests wait for the socket file descriptor to be ready for reading, with some irrelevant differences. When `fetch_ms` returns a value, we then sleep for that amount of milliseconds.

We will soon see how async functions `fetch_ms` and `sleep` can be implemented.

```python
async def func():
  ms = await fetch_ms()
  print("Will sleep now")
  await sleep(ms / 1000)
  print("Good morning")
  return "Return value"

loop = asyncio.new_event_loop()

loop.create_task(func())
loop.run_forever()
```

Instead of `asyncio`, we will add `create_task` to our simple `EventLoop` from the [previous example](/2025/01/16/AC-part3-how-event-loops-work.html). We will also introduce two additional classes to assist us. The `Future` class represents the future value and will be used by the special coroutine function to interact with the loop in order to implement tasks. The `Task` class represents a task, similar to a thread but implemented via coroutines and cooperative multitasking.

```python
from select import select
from time import time, sleep
from heapq import heappush, heappop

_global_loop = None


class Future:
    def __init__(self):
        self._callback = None
        self._result = None

    def set_done_callback(self, callback):
        # This is equivalent to `Promise.then` in JavaScript.
        self._callback = callback

    def set_result(self, result):
        # This is equivalent to calling `resolve` for Promise in JavaScript.
        self._result = result
        self._callback(self._result)


class Task:
    def __init__(self, coro):
        self._coro = coro

    def prepare(self):
        # Prepare the coroutine.
        future = self._coro.send(None)

        assert isinstance(future, Future), \
            "This EventLoop doesn't know what to do with yielded values that are not Future"

        future.set_done_callback(self.next_step)

    def next_step(self, result):
        try:
            # Run it till the next yield.
            future = self._coro.send(result)
            future.set_done_callback(self.next_step)
        except StopIteration:
            # We do nothing, task is finished.
            pass


class EventLoop:
    def __init__(self):
        self._ready_queue = []

        # This list will be used as a heap-based
        # priority queue, sorted by the time, so that
        # the value on top of the heap will be triggered
        # first.
        self._timer_queue = []

        # Waiters for read operation on non-blocking files to be ready.
        # We will not support writing operations in this demo.
        self._read_queue = {}

    # This is equivalent to setTimer in JavaScript.
    def call_later(self, delay, callback):
        at = time() + delay
        heappush(self._timer_queue, (at, callback))

    def call_when_ready_for_reading(self, file, callback):
        # This is analogous to register...handler in the previous example.
        # A bit more generalized, though.
        # File can be anything, stdin (keyboard), file descriptor,
        # network socket, etc.
        self._read_queue[file] = callback

        # We should also have methods to remove files from the queue,
        # but let's keep the code short and simple.

    def run_once(self):
        # Check what timers are ready.
        while len(self._timer_queue) > 0 and self._timer_queue[0][0] < time():
            _, callback = heappop(self._timer_queue)
            self._ready_queue.append(callback)

        # Check if some of the files have data to be read.
        rlist = list(self._read_queue.keys())
        ready_files = select(rlist, [], [], 0)[0]
        for rf in ready_files:
            callback = self._read_queue[rf]
            del self._read_queue[rf]
            self._ready_queue.append(callback)

        # Run ready callbacks.
        for callback in self._ready_queue:
            callback()
        self._ready_queue.clear()

    def run_forever(self):
        # Not the best way to set currently running loop, but it is OK for now.
        global _global_loop
        _global_loop = self
        while True:
            self.run_once()
            # There are better ways than this,
            # but this is simple, and keeps our CPU fans quiet.
            sleep(0.1)

    def create_task(self, coro):
        task = Task(coro)
        self._ready_queue.append(task.prepare)

```

We will also add two special coroutines that interact with the loop and provide interface that doesn't reveal anything about the loop to the caller, but the caller has to be coroutine.

One coroutine will be used to read from file in async way, using even loop under the hood. The other implements `sleep` on top of event loop. Both functions are meant to be used by coroutines to work with async IO.

```python

# Sometimes functions like these belong to the same library as the event loop.


def get_loop():
    # This is just for demo.
    # Here we already have instance of loop present in the file.
    # Otherwise, this function should know how to get the loop instance.
    # If it is written for specific library/module and specific loop
    # that should be easy.
    # See https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_running_loop
    # We should not bother ourselves how get_running_loop works right now.
    return _global_loop


def loop_read_from_file(file):
    os.set_blocking(file.fileno(), False)

    future = Future()

    def read_ready_callback():
        # Read from file and set that as the result of the future.
        # This should not block, as event loop will not call this callback
        # until there is some data ready to be read, and we have set the file
        # to non-blocking mode too.
        future.set_result(file.read())

    loop = get_loop()
    loop.call_when_ready_for_reading(file, read_ready_callback)

    # First we yield the future. This will yield all the way to the initial
    # caller like in the example with "d <-> c <-> b <-> a" coroutines,
    # where a yield directly to the initial caller. Here initial caller will be
    # the loop it self. It will send us the result, and we will return the
    # result to the coroutine one level up (see the mentioned example above).

    # This will send the future to the loop.
    # It is also the place where current coroutine suspends, and event loop
    # gets the control.
    result = yield future

    # This will send result to the caller coroutine.
    return result


def loop_sleep(delay):
    loop = get_loop()
    future = Future()

    def callback():
        # No special result from sleep needed.
        future.set_result(None)

    loop.call_later(delay, callback)

    yield future

    return  # Not needed, actually.

```

Finally, we have code that concurrently waits for 5 seconds to print "hello", and runs task that waits for user to press Enter, and then sleeps for 3 seconds, printing message before and after the sleep.

```python

def fetch_ms():
    # This will wait until there is something to read from stdin.
    # Usually this means we will wait until user presses key on keyboard (+Enter).
    # We ignore the result, we just want to wait until user presses the Enter.
    yield from loop_read_from_file(sys.stdin)

    # Waiting above simulated waiting for server response.
    # We mock the result with nice value of 3 seconds.
    return 3000


def func():
    ms = yield from fetch_ms()
    print("Will sleep now")
    yield from loop_sleep(ms / 1000)
    print("Good morning")
    return "Return value"


def print_hello():
    print("hello")


loop = EventLoop()

# Print hello after 5 seconds.
loop.call_later(5.0, print_hello)

loop.create_task(func())

loop.run_forever()
```

Depending on how soon user presses the Enter key, it can print something like this:

```
Will sleep now
hello
Good morning
```

To **summarize** this section, we have shown how an event loop can execute asynchronous tasks, which are actually coroutines (a special kind of generator, which are a special kind of iterator). Execution starts from the event loop (after calling `run_forever`), but then is transferred to user-defined tasks (e.g., `func`). User-defined tasks can call multiple "subcoroutines" that do not suspend directly. Instead, those coroutines "in-between" just transfer control to other coroutines (e.g., using `yield from` or equivalent). Finally, low-level coroutines suspend (e.g., using `yield` or equivalent). When those lower-level coroutines yield, they yield a `Future` (equivalent to a `Promise`), all the way back to the event loop. Then the event loop sets the callback to be called when the promise is fulfilled, and that callback resumes the coroutine (resumes the task). This way, we have built an elegant blocking-like mechanism for I/O on top of a callback-based event loop.

## Async/await

OK, great, we have a event loop written from scratch, we have timer callback, and a coroutine task that executes concurrently. But what about async/await. Well, it is mostly just a matter of syntax. For more details see [PEP 492](https://peps.python.org/pep-0492/).

We will modify few things now to support async/await syntax.

First we need a class and a decorator that will add really simple decoration to our `loop_...` functions.

```python

class AwaitableWrapper:
    def __init__(self, generator_obj):
        self._generator_obj = generator_obj

    def __await__(self):
        # Generator objects are iterators.
        # They are also iterables, which means their __iter__ method returns self.
        # They are more than iterables, and iterators, they have `send` method.
        # But `await` syntax demand them to also have __await__ method,
        # that has to return iterator (but of course generator with `send` is even better).
        # So we provide here awaitable wrapper that will return the generator.
        return self._generator_obj.__iter__()


def as_async_coroutine(generator_func):
    def wrapped(*args, **kwargs):
        generator_obj = generator_func(*args, **kwargs)
        return AwaitableWrapper(generator_obj)

    return wrapped


@as_async_coroutine
def loop_read_from_file(file):
  # Here, code stays the same as before.
  ...

@as_async_coroutine
def loop_sleep(delay):
  # Here, code stays the same as before.
  ...
```

The decorator wraps the generator function in such way that instead of returning generator object, it returns awaitable, which has `__await__` magic method, that simply returns the `__iter__()` of the wrapped generator object (which actually just returns the generator object itself). That is it, now we can just change our functions by adding `async` to them, and replacing `yield from` with `await`. For more details see [PEP 492](https://peps.python.org/pep-0492/).

```python
async def fetch_ms():
    # This will wait until there is something to read from stdin.
    # We ignore the result, we just want to wait until user presses the Enter.
    await loop_read_from_file(sys.stdin)

    # Waiting above simulated waiting for server response.
    # We mock the result with nice value of 3 seconds.
    return 3000


async def func():
    ms = await fetch_ms()
    print("Will sleep now")
    await loop_sleep(ms / 1000)
    print("Good morning")
    return "Return value"

```

Now to **see once more a complete picture**, and what all this boils down to, we will write everything without any syntax sugar and helpers, using only raw primitives. We will only leave `async def func` as it is, with async/await to **show that it really works**.

```python
# EventLoop, Future and Task remain the same.


class LoopReadFromFileGenerator:
    def __init__(self, file):
        self.file = file
        self.state = 0

    def send(self, result):
        if self.state == 0:
            os.set_blocking(self.file.fileno(), False)

            future = Future()

            def read_ready_callback():
                future.set_result(self.file.read())

            loop = get_loop()
            loop.call_when_ready_for_reading(self.file, read_ready_callback)

            self.state = 1
            return future
        elif self.state == 1:
            raise StopIteration(result)

    def __next__(self):
        return self.send(None)

    def __await__(self):
        return self


def loop_read_from_file(file):
    return LoopReadFromFileGenerator(file)


class LoopSleepGenerator:
    def __init__(self, delay):
        self._delay = delay
        self._state = 0

    def send(self, result):
        if self._state == 0:
            loop = get_loop()
            future = Future()

            def callback():
                # No special result from sleep needed.
                future.set_result(None)

            loop.call_later(self._delay, callback)

            self._state = 1
            return future
        elif self._state == 1:
            raise StopIteration(result)

    def __next__(self):
        return self.send(None)

    def __iter__(self):
        # This method is not actually needed for our example, but usually
        # all awaitables are iterables, end __await__ is equal to __iter__.
        return self

    __await__ = __iter__


def loop_sleep(delay):
    return LoopSleepGenerator(delay)


# as fetch_ms would look like without await and yield from (simplified)
# def fetch_ms():
#     subcoro = loop_read_from_file(sys.stdin)
#     output = subcoro.send(None)
#     input = yield output
#     while True:
#         try:
#             input = yield subcoro.send(input)
#         except StopIteration:
#             break

#     return 3000

class FetchMsGenerator:
    def __init__(self):
        self._state = 0
        self._subcoro = None

    def send(self, input):
        if self._state == 0:
            self._subcoro = loop_read_from_file(sys.stdin)
            output = self._subcoro.send(None)
            self._state = 1
            return output
        elif self._state == 1:
            try:
                return self._subcoro.send(input)
            except StopIteration:
                self._state = 2
                raise StopIteration(3000)

    def __next__(self):
        return self.send(None)

    def __await__(self):
        return self


def fetch_ms():
    return FetchMsGenerator()


async def func():
    ms = await fetch_ms()
    print("Will sleep now")
    await loop_sleep(ms / 1000)
    print("Good morning")
    return "Return value"


def print_hello():
    print("hello")


loop = EventLoop()

loop.call_later(5.0, print_hello)

loop.create_task(func())

loop.run_forever()

```

This works the same, but from the outside. Inside, this is much less performant, writing generators via generator functions produce optimized code. It is just one way to implement event loop for coroutines. In Python, there is no single default event loop that everything runs on. We have implemented our own event loop here, there is standard even loop from `asyncio` [module](https://docs.python.org/3/library/asyncio.html), and there is also `trio`. [Trio](https://trio.readthedocs.io/en/stable/) is not based on callbacks at all, instead it only operates with coroutine tasks (later, we will see why this is important). One more thing to note is that this code is simplified, we have used `send` but haven't used `throw` method to raise exception inside coroutine. All examples are greatly simplified to make them easier to understand. Still, these examples illustrate what happens behind the scenes with async/await, and event loops, and the principles it works on.

Another complementary material on how async/await works in Python is [this](https://tenthousandmeters.com/blog/python-behind-the-scenes-12-how-asyncawait-works-in-python/) great article by Victor Skvortsov.

Next, we will take a look at another type of coroutines, called greenlets. After that we will see what are stackless and stackfull coroutines by look at examples in C++.