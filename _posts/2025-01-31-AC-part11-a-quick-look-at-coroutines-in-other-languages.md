
---
layout: post
author: kormang
---

We've covered basically all the different ways coroutines can be implemented. Every other implementation is just something that is a combination of those, either from the underlying mechanism perspective or the syntactic perspective.

In the Go language, coroutines do not have any special syntax; they are just normal functions that can be started in another coroutine (light thread, fiber) using the `go` statement, e.g., `go func()`. So they are similar to stackful coroutines in C++.

In Kotlin, coroutines are more similar to stackless coroutines from C++20 when it comes to implementation, but they don't have any `await` keyword, and instead, they do not return awaitable objects (which makes them similar to stackful coroutines from C++, but the reason they don't return awaitable objects is probably structured concurrency which we will talk about later). Kotlin coroutines can be suspended on one thread and resumed on another (just like C++20 coroutines). They can run on a thread pool or on top of the Async IO Event Loop or on top of traditional GUI Event Loops ([see examples](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md)).

Java also got coroutines named `Virtual Threads` as part of the so-called Project Loom. Those work similarly to stackful coroutines in C++ (e.g., Boost). For more information about the difference between Java's Virtual Threads and Kotlin's coroutines, it is recommended to watch the video [Coroutines and Loom behind the scenes by Roman Elizarov](https://www.youtube.com/watch?v=zluKcazgkV4), the creator of Kotlin coroutines.