
## Conclusion

We have explored the roots of concurrency from the everyday physical world, then taking a short look at hardware-level concurrency, and then climbing up through the levels of abstraction all the way to distributed systems. We've seen that there are two foundational approaches to I/O, one based on events and callbacks, and the other based on polling. We've also seen that we can build one approach on top of the other. We have also seen the third approach which has to work on top of one of the two foundational approaches. The third approach implies spawning more than one thread or coroutine, or anything similar that suspends when waiting for I/O. We have seen that it is the preferred approach when it comes to reliability and readability if control flow changes with time, but can have performance implications depending on the underlying implementation, and can be more complicated if control flow is fixed.

We've seen more specific and advanced approaches such as Reactive Programming, and Structured Concurrency.

There is much more to explore when it comes to concurrency, but hopefully, this material should give a basic overview of the subject and how different concepts work and how they relate to each other.

That is it for now. I may write about more topics related to concurrency in the future, or analyze some of the already covered topics in more details.
