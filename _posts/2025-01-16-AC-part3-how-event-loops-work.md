---
layout: post
author: kormang
---

## Non-blocking IO principles

One of the highly concurrent environments are web servers. Web servers serve a large number of requests simultaneously. To serve those requests simultaneously, many web servers in the early days of the web were using the thread-per-request model. This implies starting a new thread for each new request, and the new thread will process data from the request and assemble a reply.

Here is a typical example of such code in Java.

```java
ServerSocket serverSocket = new ServerSocket(8080);
System.out.println("Server listening on port");

// Keep listening for connections in a loop.
while (true) {
    // Wait (block) until we accept incoming connection.
    Socket clientSocket = serverSocket.accept();
    System.out.println("Accepted connection from " + clientSocket.getInetAddress());

    // Spawn a new thread to handle the connection
    Thread thread = new Thread(new ConnectionHandler(clientSocket));
    thread.start();
}
```

This model has a few drawbacks. Threads consume relatively large amounts of memory and are relatively slow to start. To solve the second problem, thread pools of reusable threads are often used. With too many threads, context switches between them that the OS and CPU perform are also costly; CPU registers need to be saved, a user-space to kernel-space switch has to be performed, and it is cache-unfriendly. Another problem is that each thread calls blocking system calls to perform IO operations. We've just mentioned how that actually makes code much easier to follow, with linear control flow. However, it is also relatively slow. System calls require a switch between user space and kernel space, and a lot of maintenance work inside the kernel to manage all the threads and what IO operations they are waiting for.

So, another pattern appeared. OS kernels adopted non-blocking IO APIs. On Linux, these are the `select`, `poll`, and `epoll` system calls.

The idea behind these APIs is not to wait for a specific IO operation and block the current thread, but to ask the OS if there are any IO operations that we're interested in that are completed.

To simplify things and at the same time illustrate the concept, we'll take a look at pseudo code that uses an (imaginary) function `pull` from the OS to get the completed IO operations.

```c
while (true) {
  // Check if any IO is completed.
  io_op = pull();

  if (is_keyboard_event(io_op)) {
    KeyboardInputData kbdata = io_op.data.as_kbdata();
    char c = kbdata.character;
    do_something_with_the_char(c);
  } else if (is_tcp_data_received(io_op)) {
    TcpDataReceived = tcp_data_rec = io_op.data.as_tcp_data_recv();
    // for example
    write_to_file(opened_file, tcp_data_rec.bytes);
    // The above line will also trigger another io
    // operation to complete asynchronously and will not
    // block current thread.
  } else if (is_empty(io_op)) {
    // No IO operation has completed.
    do_something_else();
    // After we do something else, we go back to the
    // start of the loop, and check again if there is
    // any IO operation completed.
  }
}
```

However, usually we don't put all IO handling code in one loop. Instead we register handlers, and from the loop we call those handlers when there is new IO operation completed.

```c
while (true) {
  // Check if any IO is completed.
  io_op = pull();

  if (not_empty(io_op)) {
    handler = find_registered_handler_for_op(io_op)
    // call handler
    handler(io_op.data)
  } else {
    // No IO ready yet, but we can continue doing something else.
    do_something_else_for_short_period_of_time();
  }
  // Now repeat the process, see if there is any IO completed.
}
```

So, this is the polling approach, but let's see how it looks from the perspective of a handler. Again, to simplify things, we will use pseudo code.

```c
void keyboard_handler(IoOpData data) {
  KeyboardInputData kbdata = cast(data);
  char c = kbdata.character;
  do_something_with_the_char(c);
}

register_keyboard_handler(keyboard_handler);

void tcp_connection_data_received(IoOpData data) {
  TcpDataReceived = tcp_data_rec = data.as_tcp_data_recv();
  write_to_file(opened_file, tcp_data_rec.bytes);
}

register_tcp_data_received_handler(tcp_connection, tcp_connection_data_received);
```

This code is extremely simplified, but if you imagine that we have many handlers, and each of them performs more complex tasks that are omitted here, then it is not hard to see that we've moved from the polling approach back to the event handling approach. Here, events can be a keyboard press, or data received over a TCP connection. It can also include a new TCP connection being accepted, or data successfully written to disk, and so on.

## Event loop with polling in Python

We will create an event loop in Python. The event loop will be able to call timers (similar to the `setTimeout` function in JavaScript) and will be able to call callbacks when there is data to be read from a file (a file in Unix generalized terms, meaning stdin, network socket, or regular file).

We will use the `select` system call to poll for files that are ready for reading.
Select is one of the polling system calls available on Linux ([docs](https://man7.org/linux/man-pages/man2/select.2.html)), and we will use Python wrapper that comes with Python installation ([docs](https://docs.python.org/3/library/select.html)). This is thin wrapper around native C-API of Linux kernel, directly presented to Python programs.

For timers, we will use priority queue based on heap data structure (list ordered in a special way to support fast popping of the least element O(1), and fast insertions of new elements O(log(n))).

```python
from select import select
from time import time, sleep
from heapq import heappush, heappop


class EventLoop:
    def __init__(self):
        self._ready_queue = []

        # This list will be used as a heap-based
        # priority queue, sorted by the time, so that
        # the value on top of the heap will be triggered
        # first.
        self._timer_queue = []

        # Waiters for read operation on nonblocking files to be ready.
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
            self._ready_queue.append(callback)

        # First we've added timer callbacks to ready queue.
        # Then we've added callbacks for read operations for which OS says they
        # are ready (we asked OS what files are ready by calling select).
        # Finally call those callbacks that are ready.
        for callback in self._ready_queue:
            callback()
        self._ready_queue.clear()

    def run_forever(self):
        while True:
            # Run one cycle of the event loop.
            self.run_once()

            # Optionally sleep for some time.
            # There are better ways than this,
            # but this is simple, and keeps our CPU fans quiet.
            # sleep(0.1)

            # In GUI systems after running one cycle of the event loop,
            # rendering of the GUI to screen is usually performed.

```

That is it, that is the event loop. Now we can use it like this.

```python
import sys
import os
from .event_loop import EventLoop

loop = EventLoop()


def print_hello():
    print("Hello")


def print_input():
    # This read should never block, waiting for data
    # It should always return immediately, since `select` "said" there
    # are data to be read.
    print("input:", sys.stdin.read())


# Print hello after 5 seconds.
loop.call_later(5.0, print_hello)

# Make stdin (keyboard) non-blocking.
# Note that it doesn't make it non-buffered, so you'll need to
# press Enter key to flush the buffer.
os.set_blocking(sys.stdin.fileno(), False)
loop.call_when_ready_for_reading(sys.stdin, print_input)

loop.run_forever()
```

This event loop takes a polling approach to non-blocking IO provided by the OS kernel and offers an event-based non-blocking interface to it. It also provides its own mechanism for timers.

Although this is a really simple event loop, it should give some idea about how polling and event loops work.

## GUIs and event loops

In traditional desktop GUI systems (Win32, GTK+, Qt, Swing, and so on) as well as mobile GUI systems like Android, there is one single thread that runs the event loop, similar to the above one. The difference is that in GUI systems we usually don't listen to low-level events like keyboard presses or mouse clicks, but to higher-level events, like "that specific button was clicked" or "that specific input field has changed".

In such cases, if we wanted to listen to a click on a button, we would register a handler to handle the click on that button. If the handler performs a short-lived operation, there are no problems with that approach, but let's suppose that we want to download a file when a user clicks the button. Traditionally, programmers would use blocking operations to make an HTTP (TCP) request and read data as it is received. However, this means that the handler will not finish until the file is downloaded, and it could take a long time. This implies that no other handler can execute during that time, since we have only one thread, one event loop, and we process events one by one. Usually, the rendering of the UI components to the screen is also done in that same thread after processing events and calling handlers. That means that we cannot have progress bars, and we cannot click on a button that would cancel the download. The traditional approach to solving this kind of problem is using a background worker thread to download the file. Now the main thread that processes IO events and renders GUI and the background worker thread that downloads the file can do their job at the same time, concurrently. The background worker thread would send events about progress to the main UI thread to update the progress bar, and the main UI thread would send a cancel event to the background worker thread from the click handler of the "Cancel" button.

As we can see, GUIs are highly concurrent environments, and traditionally concurrency is handled using event handlers, and IO operations and other long-running operations are performed in background worker threads.

But there is an alternative approach. JavaScript is a language originally made for GUI programming inside web pages. JavaScript code has to handle a lot of UI events, but also a lot of IO operations (downloads, uploads, and so on). For that purpose, JavaScript has a built-in event loop. In JavaScript we don't use threads for upload and download; instead we use asynchronous APIs to perform those kinds of IO operations. This is much simpler then using background worker threads.

If we have a button on our web page, and the button has the ID "btn1", then we can register a handler for a click on that button like this:

```javascript
document.getElementById("btn1").addEventListener("click", makeRequest);
```

Handlers are called listeners in JavaScript, but are the same things. When click event happens, it will call function `makeRequest`.

```javascript
function makeRequest() {
  var xhr = new XMLHttpRequest();

  // GET-request to the URL /api/data
  xhr.open('GET', '/api/data', true);

  // Set up a function that will be called when the request is completed
  xhr.onload = function() {
    if (xhr.status >= 200 && xhr.status < 300) {
      // Request was successful, handle the response
      console.log('Success! Response:', xhr.responseText);
    } else {
      // Request failed, handle the error
      console.error('Request failed with status:', xhr.status);
    }
  };

  // Set up a function that will be called if there's an error with the request
  xhr.onerror = function() {
    console.error('Network error occurred');
  };

  // Send the request, which will be completed asynchronously, and upon completion
  // onload function above will be called.
  xhr.send();
}

```

This kind of response will not block the UI thread; instead, the `onload` function will be called from the event loop when the corresponding I/O operation finishes.

This also allows us to make two HTTP requests at once, and even set a timer at the same time to (let's say) perform some kind of animation while the download is in progress (note that using timers for animations is usually, but not always, a bad idea).

This gave birth to Node.js, a JavaScript runtime without a GUI, but with an event loop used exclusively for I/O operations, for server-side code. Servers perform a lot of I/O operations, and such architecture has proved to be highly effective at utilizing the full potential of CPUs while performing many I/O operations concurrently.

Next we will see the pros and cons of using the event handler approach and callbacks.
