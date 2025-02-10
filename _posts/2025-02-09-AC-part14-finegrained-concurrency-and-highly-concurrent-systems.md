---
layout: post
author: kormang
---

One of the reasons why concurrency with threads and with callback-based I/O is difficult is that it makes it either inconvenient or inefficient to start new concurrent operations at any point in time.

But why would we do that at all?

Consider this algorithm.

![Algorithm sequential](/assets/images/algorithm_seq.png "Algorithm sequential")

If the condition is met, we download one file, then the other, then we process one file, then the other, and again we check if the condition is met.

In case downloading and processing those files is needed, and if it can be done at the same time, we could start two concurrent operations. Downloading is easy to do at the same time, as the CPU is mostly waiting for data, so two lightweight coroutines could be started easily. These two coroutines could start the download and wait until it is done.

For the purpose of this example, let's say that processing the file cannot start before both are downloaded, but it can be done in parallel, once both are downloaded. Also, let's say that processing data from those files is a CPU-intensive operation.

Since processing those files is a CPU-intensive operation, it will require starting two OS threads to utilize two CPU cores.

Starting two coroutines on demand whenever needed is easy and cheap; they are efficient. Starting and shutting down two OS threads is relatively expensive, complicated, and slow. In case it is not inconvenient (we use a library that makes it easy) and processing that data is really slow so that starting OS threads isn't noticeable compared to the work done by those threads, then we could really start two OS threads and shut them down on demand.

This is the new algorithm we get.

![Algorithm parallel](/assets/images/algorithm_par.png "Algorithm parallel")

Usually, starting OS threads like that is not a good option. To achieve the same thing, some environments and languages offer thread pools. Thread pools contain threads that are waiting for tasks to be assigned to them and then execute those tasks in parallel. After they finish tasks, they don't shut down; they wait for another task.

Some environments and languages offer the possibility to run coroutines on different threads so we can start two coroutines to process data from files. However, those two coroutines are not run on top of a single OS thread and event loop; instead, they are sent to two different worker threads to execute in parallel.

Another option is to wrap the task in some other form and send it to the worker threads, then start two coroutines that wait for the results to be returned from the worker threads. These coroutines can now run in parallel on their main thread because both will be suspended almost all the time. We could also use worker processes instead of threads. In such a case, tasks and the data for the tasks are sent to another process to be processed in isolation. We could also send them to a completely different machine, one that, for example, has special hardware that can process these tasks faster. That involves new I/O operations, and copying/sending data to another process/machine. So we could look at sending tasks to worker threads as I/O operations that we wait for in coroutines, but without actually copying and sending data.

The ability to start and stop new concurrent operations easily, on demand, is a really powerful concept. However, without structured concurrency, it is equivalent to using `goto` everywhere. Starting a new concurrent operation could be as dangerous as a `goto` that jumps to a random place without cleaning up resources first.
