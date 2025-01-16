---
layout: post
author: kormang
---

## Good use cases for event handlers approach

If we have event `E1` and event handler for that event that always executes operations `OpA`, `OpB` and `OpC`, and event `E2` with event handler that always executes operations `OpD` and `OpE`, then event handlers (callback) might be a good fit for the job. In fact, callbacks might be perfect fit for the job. If operations are always the same, and do not change (_a lot_) depending on the state of the program, and the operations don't change the state themselves (_a lot_) then it is perfect fit for the event handler based asynchronous programming. In other words, if program can be described like this: "Whenever _this_ happens just do _that_", then event handlers or callbacks are good fit for the job.

This is often the case with web servers. In the core of the web server there might be some polling based mechanism that waits for new request. When new request arrives, the server framework will check the method and the path of the HTTP request and decide what handler to call.

```python
# Define handlers.
def handle_get_contacts(req):
  return as_json(find_contacts(req.user_id))

def handle_post_new_contact(req):
  return as_json(add_contact_to_user(req.user_id, req.contact_id))

# Register handlers for GET and POST HTTP methods.
server.add_handler("GET", "/contact/", handle_get_contacts)
server.add_handler("POST", "/contact/", handle_post_new_contact)


# Main server loop.
while True:
  req = server.wait_for_new_http_request()
  handler = find_handler_for_method_and_path(req.method, req.path)

  # Call handler directly
  handler(req)
  # or execute it in different thread, or some other execution environment to
  # enable concurrent requests.
  execute_handler_in_thread_pool(handler, req)

```

_Thread pools are just sets of threads ready to perform some task._

This way we have switched from polling to event handler based mechanism.

This is good fit for callbacks because every time there is `GET` request to `/contact/` we do the same thing, as well as for `POST` request to `/contact/`. This way we can write handlers for different endpoints independently and scale our code base easily, as handlers are independent one from another.

Usually in such backend applications **there is common state that is mutated** by these handlers - **database**. However, it rarely affects execution of the handlers, as it is rarely used to store state of the state machine that represents program (few flags here and there are not that significant).

In one of the next posts we will see when HTTP endpoint handlers can have a lot of shared state, when **database becomes state of the state machine**, and when implementing such logic using event handlers for HTTP endpoints can become complicated.

It is similar with GUIs. GUIs are often good fit for callback based mechanism - e.g. "whenever user clicks this button add contents of that input field to that list", but there are also cases when callbacks start relying too much on state as in state machine, and it is easier to write GUIs using different approaches.

## Bad use case for event handlers approach

In the first post of this series about concurrency we had already an [example](./2025/01/12/AC-part2-layers-of-abstraction.html#application-using-event-handlers) when writing logic using event handler(s) can lead to very hard to read code.

Here we will write shorter example involving networking in python-like pseudo code.

```python
last_error = None
data1 = None
data2 = None

def on_error(error):
  global last_error
  last_error = error

def final_processing():
  if last_error is not None:
    process_data(data1, data2)

def on_data(data):
  global data1
  global data2
  if last_error is not None:
    close_connections()
    return
  if data1 is None:
    data1 = data
    fetch_data("/data2")
  else:
    # This means data is actually data2.
    data2 = data
    close_connections()


register_on_error_handler(on_error)
register_on_connections_closed_handler(final_processing)
register_on_data_fetched(on_data)
fetch_data("/data1")
```

This is super simple example, but it should be enough to see why this is not the most maintainable code possible. The reason is global state, that represents state machine. We could avoid state machine if our API is a bit different, if functions accepts event handling functions, so we don't have to use `register_` functions for everything. This way, less of the global state is explicitly shared state machine state.

```python
last_error = None
data1 = None
data2 = None

def on_error(error):
  global last_error
  last_error = error

def final_processing():
  if last_error is not None:
    process_data(data1, data2)


def on_data2(data):
  global data2
  data2 = data
  if last_error is None:
    close_connections(final_processing)

def on_data1(data):
  global data1
  data1 = data
  if last_error is None:
    fetch_data("/data2", on_data2)
  else:
    close_connections(final_processing)


register_on_error_handler(on_error)
fetch_data("/data1", on_data1)

```

We can keep callback based approach by using closures, to further simplify code.

```python
def do_nothing():
  pass

def on_data1(error, data1):
  if error is None:
    def on_data2(error, data2):
      if error is None:
        def final_processing(error):
          if error is None:
            process_data(data1, data2)
        close_connections(final_processing)
    fetch_data("/data2", on_data2)
  else:
    close_connections(do_nothing)

fetch_data("/data1", on_data1)
```

Although this version uses callbacks, those are not entirely "event handlers".

Some might find this easier to navigate as there is no global state, but it is still obviously far from readable and maintainable.

Using inline function (which Python has only in form of single line lambdas) as in JavaScript it would look like this:

```javascript
function do_nothing() {

}

function on_data1(error, data1) {
  if (!error) {
    fetch_data("/data2", function(error, data2) {
      if (!error) {
        close_connections(function(error) {
          if (!error) {
            process_data(data1, data2)
          }
        })
      }
    })
  } else {
    close_connections(do_nothing)
  }
}

fetch_data("/data1", on_data1)
```

One could argue that this is even simpler to read and maintain, because we don't have to define function before we pass them as arguments, and instead we can define them inline, which kind of makes it look similar to how it would look like in "normal" case, just indented.

That "normal" case would require functions that are waiting for async operations to complete and return result, or throw an exception in case of an error.

The normal code would look like this:

```python
closing_connections = False
try:
  data1 = fetch_data("data1")
  data2 = fetch_data("data2")
  # In all of the examples above we process data after closing connections, but also in case on an error.
  closing_connections = True
  close_connections()
  process_data(data1, data2)
finally:
  if not closing_connections:
    close_connections()

```

It should be obvious that this is much more maintainable code then the previous versions.

To have such functions as `fetch_data` that wait until data is fetched we have to either rely on OS blocking IO operations (syscalls) as briefly explained [here](./2025/01/10/intro-to-concurrency.html#a-bit-of-os-theory), or on user space coroutines (in Python and JavaScript those are async functions), which will be explained in-depth in the next few posts.

## Other problems with callbacks

We already saw how it can be difficult to write and read code that uses callbacks to control execution, when there is need for manual state machine management. In JavaScript, this issue is partially solved by using closures. Closures are functions that can access variables that were in the scope when the closure was instantiated. Closures can be called later as callbacks (or event handlers, or event listeners), but will still have access to variables that were present at the moment they were instantiated.

However, there are more problems with callbacks. One example is that event handlers for IO operations could make the code look like this.

```javascript
const fs = require('fs');

// Step 1: Read data from a file
fs.readFile('data.txt', 'utf8', (err, fileData) => {
  if (err) {
    console.error('Error reading file:', err);
    return;
  }

  // Step 2: Process the file data
  processData(fileData, (processedData) => {
    // Introduce additional nesting with setTimeout
    setTimeout(() => {
      console.log('Processing data asynchronously...');

      // Step 3: Make an API call
      apiCall(processedData, (apiResult) => {
        // Introduce more nesting with setTimeout
        setTimeout(() => {
          console.log('Making API call asynchronously...');

          // Step 4: Perform another async operation
          anotherAsyncOperation(apiResult, () => {
            // Introduce further nesting
            setTimeout(() => {
              console.log('Performing another async operation...');

              // Step 5: Log the final result
              console.log('Final Result:', apiResult);
            }, 1000);
          });
        }, 1000);
      });
    }, 1000);
  });
});

function processData(data, callback) {
  // Simulate asynchronous processing
  setTimeout(() => {
    const processedData = data.toUpperCase();
    callback(processedData);
  }, 1000);
}

function apiCall(data, callback) {
  // Simulate an asynchronous API call
  setTimeout(() => {
    const apiResult = `API Result for ${data}`;
    callback(apiResult);
  }, 1000);
}

function anotherAsyncOperation(data, callback) {
  // Simulate another asynchronous operation
  setTimeout(() => {
    console.log('Executing another async operation...');
    callback();
  }, 1000);
}

```

Thanks to closures, there is no need for manual state management, but it is still hard to follow all those callbacks and nested closures. This is usually referred to as callback hell.

Besides pure readability, another problem is the lack of a proper stack trace. Consider this example.

```javascript
const http = require('http');
const fs = require('fs');

// Read the contents of the file
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error reading file:', err);
    return;
  }

  // Prepare the request options
  const options = {
    hostname: 'example.com',
    port: 80,
    path: '/upload',
    method: 'POST',
    headers: {
      'Content-Type': 'application/json', // Set the appropriate content type
    },
  };

  // Create the request
  const req = http.request(options, (res) => {
    let responseData = '';

    // A chunk of data has been received.
    res.on('data', (chunk) => {
      responseData += chunk;
    });

    // The whole response has been received.
    res.on('end', () => {
      if (res.statusCode === 200) {
        console.log('Response from server:', responseData);
      } else {
        console.error('Unexpected response status code:', res.statusCode);
        throw new Error('Unexpected response status code');
      }
    });
  });

  // Handle errors in the request
  req.on('error', (error) => {
    console.error('Error in request:', error);
  });

  // Write data to the request body
  req.write(data);

  // End the request
  req.end();
});

```

It will open file "data.txt" and try to upload it to "example.com/upload", since there is not such address, it will get the error. When we throw the exception, here is the stack trace that we get:

```
Unexpected response status code: 404
/home/user/index.js:37
        throw new Error('Unexpected response status code');
        ^

Error: Unexpected response status code
    at IncomingMessage.<anonymous> (/home/user/index.js:37:15)
    at IncomingMessage.emit (node:events:539:35)
    at endReadableNT (node:internal/streams/readable:1345:12)
    at processTicksAndRejections (node:internal/process/task_queues:83:21)
```

Indeed, not very helpful. This is because the final callback was called from the event loop, which processes, as the stack trace suggests, `task_queues`. How we arrived to the point of error is hard to tell, even in this simple example, but in real-world code, it can be much more difficult.

To solve the callback hell problem `Promises` were introduced:

```javascript
import fetch from 'node-fetch';
import fs from 'fs';
import {promisify} from 'util';

// Promisify the readFile function
const readFileAsync = promisify(fs.readFile);

// Function to perform the HTTP request and handle the response
function uploadData(data) {
  return fetch('http://example.com/upload', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ data }),
  })
  .then(response => {
    if (!response.ok) {
      throw new Error(`Failed to upload data. Status: ${response.status}`);
    }
    return response.json();
  });
}

// Read the file and upload data using promises
readFileAsync('data.txt', 'utf8')
  .then(data => uploadData(data))
  .then(response => {
    console.log('Upload successful:', response);
  })
  .catch(error => {
    console.error('Error:', error.message, error.stack);
  });

```

But the problem with the control flow, and stack still remains, this is the error we get:

```
Error: Failed to upload data. Status: 404 Error: Failed to upload data. Status: 404
    at file:///home/user/indexp.mjs:23:13
    at processTicksAndRejections (node:internal/process/task_queues:96:5)
```

To solve that problem and make code even more simpler to follow, `async` functions where introduced that return promises implicitly, and can `await` for promise to resolve.

```javascript
import fetch from 'node-fetch';
import fs from 'fs';
import { promisify } from 'util';

// Promisify the readFile function
const readFileAsync = promisify(fs.readFile);

// Function to perform the HTTP request and handle the response
async function uploadData(data) {
  const response = await fetch('http://example.com/upload', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ data }),
  });

  if (!response.ok) {
    throw new Error(`Failed to upload data. Status: ${response.status}`);
  }

  return response.json();
}

// Read the file and upload data using async/await
async function main() {
  try {
    const data = await readFileAsync('data.txt', 'utf8');
    const response = await uploadData(data);
    console.log('Upload successful:', response);
  } catch (error) {
    console.error('Error:', error.message, error.stack);
  }
}

// Call the main function
await main();
```

Now code is much more readable, it is shorter, and we can follow control flow naturally.

This is the stack trace we get:

```
Error: Failed to upload data. Status: 404 Error: Failed to upload data. Status: 404
    at uploadData (file:///home/user/indexa.mjs:19:11)
    at processTicksAndRejections (node:internal/process/task_queues:96:5)
    at async main (file:///home/user/indexa.mjs:29:22)
    at async file:///home/user/indexa.mjs:37:1
```

Not ideal, but much better.

One more big advantage of async/await approach is that **exceptions propagate naturally**, which is often overlooked but very important aspect of making software as reliable as possible. Exceptions are not the only, and maybe not the best way to handle errors, but with callbacks it is often difficult and not always obvious to whom to send error signal and how, and what **resources need to be cleaned up** along the way.

In the next few posts you will see how async function and coroutines are implemented in different programming languages.
